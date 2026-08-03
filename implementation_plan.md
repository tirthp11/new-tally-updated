# Memory Leak Root-Cause Analysis & Fix Plan

## Problem Statement

Production RAM usage grows monotonically with cumulative user actions and never reclaims, even during zero-request idle periods. This is a classic unbounded-retention pattern — objects are being created on each action and never released.

## Root Causes Identified

I identified **5 distinct memory leak sources** across the backend and **2 contributing patterns** on the frontend. They are ranked by severity (estimated RAM impact × frequency of trigger).

---

## 🔴 Critical — Backend

### Leak 1: `ConnectorHub._post_locks` — Unbounded dict, never evicted

**File:** [connector_hub.py](file:///c:/Users/Administrator/Downloads/new-tally-main/cloud/backend/app/services/connector_hub.py#L25-L33)

```python
self._post_locks: dict[str, asyncio.Lock] = {}

def post_lock(self, connector_id: str) -> asyncio.Lock:
    lock = self._post_locks.get(connector_id)
    if lock is None:
        lock = asyncio.Lock()
        self._post_locks[connector_id] = lock
    return lock
```

**Why it leaks:** Every unique `connector_id` that ever pushes a voucher creates an `asyncio.Lock` that is **never removed**, even after the connector disconnects and is deleted. Over time, with connector re-pairings (each creating a new UUID), this dict grows without bound. While each Lock is small (~100 bytes), the connector_id strings and dict overhead accumulate over months.

**Severity:** Medium-low in isolation, but the pattern is the template for the much worse Leak 2.

---

### Leak 2: `ConnectorHub._connections` — Entry removed, but detached `Connector` ORM objects held indirectly via WebSocket closures

**File:** [connector_ws.py](file:///c:/Users/Administrator/Downloads/new-tally-main/cloud/backend/app/api/v1/connector_ws.py#L194-L283)

**Why it leaks:** When `connector_ws()` runs, it loads a `Connector` ORM object from a **manually-created session** (line 207–218) which is closed immediately. However, the `connector_id = connector.id` capture (line 225) is harmless — the real problem is the **fire-and-forget `asyncio.create_task()`** calls:

- Line 232: `asyncio.create_task(_post_connect(connector_id))`
- Line 255: `asyncio.create_task(_sync_and_reply(connector_id, websocket))`
- Line 260–261: `asyncio.create_task(_sync_company_and_reply(...))`

Each of these tasks creates **its own `db = get_sessionmaker()()`** session, loads ORM objects, and closes the session in a `finally` block. The tasks themselves are fire-and-forget — if the WebSocket disconnects before the task completes, the task reference is held by the event loop but **the task's local scope holds references to the loaded ORM objects and the `websocket` object**. Until the task finishes (or the event loop is torn down), these objects remain pinned in memory.

**Crucially**, `expire_on_commit=False` is set in [session.py L34](file:///c:/Users/Administrator/Downloads/new-tally-main/cloud/backend/app/db/session.py#L34), which means every ORM object loaded in those sessions retains its full attribute state in-memory indefinitely — the objects never "go lazy" and never release their `__dict__` data even after the session is closed.

**Severity:** **HIGH**. Each `sync_and_reply` / `sync_company_and_reply` task loads the full `Connector` object, and `sync_connector` can load all `TallyCompany` rows + trigger `store_masters` which creates hundreds of `MastersCache` ORM objects. With `expire_on_commit=False`, these detached objects hold their full column data indefinitely in the task's local scope.

---

### Leak 3: OpenAI client re-instantiated on every AI call — never pooled

**File:** [extractor.py](file:///c:/Users/Administrator/Downloads/new-tally-main/cloud/backend/app/services/extraction/extractor.py#L21-L25)

```python
def _client() -> OpenAI:
    settings = get_settings()
    if not settings.openai_api_key:
        raise RuntimeError(...)
    return OpenAI(api_key=settings.openai_api_key)
```

**Why it leaks:** Every call to `call_ai_vision()` or `call_ai_text()` instantiates a **new `OpenAI()` client**. The `openai` SDK client creates an internal `httpx.Client` with its own connection pool, SSL contexts, and transport objects. These are heavyweight (~50–200KB each). If `httpx` does not eagerly close connections when the client goes out of scope (which it doesn't — it relies on `__del__` or explicit `.close()`), the underlying TCP sockets and SSL buffers linger until GC runs a finalizer cycle.

With `asyncio.to_thread(build_voucher_xml, ...)` and `asyncio.to_thread(build_masters_xml, ...)` running these calls from worker threads, GC pressure is lower and finalization is delayed.

**Severity:** **HIGH**. Each upload creates one client; each push creates 1-2 more (masters XML + voucher XML). At 50+ uploads/day, this is ~10MB+/day that lingers until a GC cycle collects the finalization queue.

---

### Leak 4: `_touch()` creates a new DB session on every WebSocket ping — never pooled with the WS loop

**File:** [connector_ws.py](file:///c:/Users/Administrator/Downloads/new-tally-main/cloud/backend/app/api/v1/connector_ws.py#L337-L346)

```python
def _touch(connector_id: uuid.UUID) -> None:
    db = get_sessionmaker()()
    try:
        connector = db.get(Connector, connector_id)
        ...
        db.commit()
    finally:
        db.close()
```

**Why it leaks:** `_touch()` is called on **every ping** (line 242), and the connector pings every 15 seconds (matching the frontend poll interval). Each call creates a full SQLAlchemy session, loads a `Connector` ORM object with `expire_on_commit=False`, commits, and closes. While the session is closed, the `Connector` object loaded with `expire_on_commit=False` is detached with its full attribute state. It becomes garbage-eligible only after `_touch()` returns and the local `connector` variable goes out of scope — but CPython's garbage collector doesn't necessarily collect it immediately, especially under asyncio event-loop pressure.

Worse, `_touch` is called from the sync path of the WebSocket receive loop (line 242 calls `_touch(connector_id)` directly, not via `await`). Since it creates a session inside a synchronous function called from an async context, the session lifecycle can interact poorly with the event loop.

**Severity:** Medium. Each touch is small, but at 4 pings/minute per connector × N connectors, the allocation rate is constant and substantial.

---

### Leak 5: `_persist_orphan_result()` — identical session pattern, called on unmatched job results

**File:** [connector_ws.py](file:///c:/Users/Administrator/Downloads/new-tally-main/cloud/backend/app/api/v1/connector_ws.py#L349-L368)

Same pattern as `_touch()`: creates a new session, loads a `Job` ORM object, commits, closes. With `expire_on_commit=False`, the detached ORM objects retain their full state. This is triggered by orphan results (e.g., after server restart), so it's less frequent but still cumulative.

**Severity:** Low (triggered infrequently).

---

## 🟡 Moderate — Frontend (browser-side, contributes to backend memory via never-cleaned-up polling)

### Leak 6: `statusTimer` (connector status polling) never cleared on logout/navigation

**File:** [shell.js](file:///c:/Users/Administrator/Downloads/new-tally-main/cloud/frontend/js/shell.js#L405-L427)

```javascript
let statusTimer = null;
function startStatusPolling() {
    if (statusTimer) clearInterval(statusTimer);
    statusTimer = setInterval(pollConnectorStatus, 15000);
    pollConnectorStatus();
}
```

`startStatusPolling()` is called on every login (line 709), but `statusTimer` is **never cleared on logout**. If a user logs out and back in, a second `setInterval` starts — but the first one would have been cleared by the `if (statusTimer)` check. However, the real issue is that the timer keeps firing HTTP requests to `/connector/status` even after logout (the request fails with 401 but the timer keeps running), generating unnecessary backend session creation.

**Severity:** Low-medium. Does not directly leak backend memory but contributes unnecessary session churn.

---

### Leak 7: `abs:company-changed` event listeners accumulate across SPA navigations

**Files:**
- [vouchers.js L73](file:///c:/Users/Administrator/Downloads/new-tally-main/cloud/frontend/js/vouchers.js#L66-L73)
- [history.js L106](file:///c:/Users/Administrator/Downloads/new-tally-main/cloud/frontend/js/history.js#L99-L106)
- [auth.js L82](file:///c:/Users/Administrator/Downloads/new-tally-main/cloud/frontend/js/auth.js#L75-L82)

Each of these views registers a `document.addEventListener("abs:company-changed", onCompanyChanged)` on every view init. The listeners do have a self-cleanup guard (`if (!document.body.contains(body))` → `removeEventListener`), which means they clean up **on the next company-change event** after navigating away. But if no company change happens, the old listener stays registered and its closure retains the old `state` object, `body` element, `tbody`, and all voucher data from the previous view rendering.

**Severity:** Low-medium. The closures hold references to DOM elements already removed from the tree (so the elements themselves are eligible for GC only once the closure is released). In a single-page app where users navigate many times between views, this can retain large voucher data arrays.

---

## Proposed Changes

### Priority 1 — Singleton OpenAI client (Leak 3)

#### [MODIFY] [extractor.py](file:///c:/Users/Administrator/Downloads/new-tally-main/cloud/backend/app/services/extraction/extractor.py)

Replace per-call `OpenAI()` instantiation with a module-level singleton, lazily initialized and reused:

```python
_openai_client: OpenAI | None = None

def _client() -> OpenAI:
    global _openai_client
    if _openai_client is None:
        settings = get_settings()
        if not settings.openai_api_key:
            raise RuntimeError(...)
        _openai_client = OpenAI(api_key=settings.openai_api_key)
    return _openai_client
```

**Impact:** Eliminates ~50–200KB per AI call. Single largest per-action memory saving.

---

### Priority 2 — Evict stale entries from `_post_locks` (Leak 1)

#### [MODIFY] [connector_hub.py](file:///c:/Users/Administrator/Downloads/new-tally-main/cloud/backend/app/services/connector_hub.py)

Clean up a connector's `_post_locks` entry in `unregister()`:

```python
async def unregister(self, connector_id: str, ws: WebSocket) -> None:
    removed = False
    async with self._lock:
        if self._connections.get(connector_id) is ws:
            del self._connections[connector_id]
            removed = True
    if removed:
        self._post_locks.pop(connector_id, None)   # <-- NEW
        log.info("Connector offline: %s", connector_id)
```

**Impact:** Caps `_post_locks` at the number of *currently online* connectors.

---

### Priority 3 — Fix fire-and-forget task retention (Leak 2)

#### [MODIFY] [connector_ws.py](file:///c:/Users/Administrator/Downloads/new-tally-main/cloud/backend/app/api/v1/connector_ws.py)

Store references to background tasks and cancel them on disconnect:

```python
# In connector_ws():
tasks: set[asyncio.Task] = set()

def track(coro):
    task = asyncio.create_task(coro)
    tasks.add(task)
    task.add_done_callback(tasks.discard)
    return task

# Replace:   asyncio.create_task(_post_connect(connector_id))
# With:      track(_post_connect(connector_id))
# Same for _sync_and_reply and _sync_company_and_reply.

# In the finally block, cancel all outstanding tasks:
finally:
    for t in tasks:
        t.cancel()
    await hub.unregister(str(connector_id), websocket)
    ...
```

**Impact:** Ensures ORM objects loaded in background tasks are released promptly when the WebSocket closes. Prevents accumulation of detached `Connector`, `TallyCompany`, and `MastersCache` objects.

---

### Priority 4 — Eliminate per-ping session churn (Leak 4)

#### [MODIFY] [connector_ws.py](file:///c:/Users/Administrator/Downloads/new-tally-main/cloud/backend/app/api/v1/connector_ws.py)

Batch `last_seen_at` updates instead of creating a session on every ping. Two approaches:

**Option A (simpler):** Throttle `_touch()` to run at most once per minute:

```python
_last_touched: dict[uuid.UUID, float] = {}

def _touch(connector_id: uuid.UUID) -> None:
    import time
    now = time.monotonic()
    if now - _last_touched.get(connector_id, 0) < 60:
        return
    _last_touched[connector_id] = now
    db = get_sessionmaker()()
    try:
        ...
    finally:
        db.close()
```

**Option B (better):** Move the touch into the `_post_connect` background task's session and do a single update when the WS closes, instead of per-ping.

**Impact:** Reduces session allocation from ~4/minute/connector to ~1/minute/connector.

---

### Priority 5 — Clear frontend polling timer on logout (Leak 6)

#### [MODIFY] [shell.js](file:///c:/Users/Administrator/Downloads/new-tally-main/cloud/frontend/js/shell.js)

Add a `stopStatusPolling()` function and call it from `ABSAuth.logout()` or `refreshChrome` when `showChrome` is false:

```javascript
function stopStatusPolling() {
    if (statusTimer) { clearInterval(statusTimer); statusTimer = null; }
}
```

In `refreshChrome`:
```javascript
if (showChrome) {
    ...
} else {
    stopStatusPolling();   // <-- NEW
    ...
}
```

**Impact:** Prevents unnecessary backend requests after logout.

---

### Priority 6 — Eagerly remove stale event listeners (Leak 7)

#### [MODIFY] [vouchers.js](file:///c:/Users/Administrator/Downloads/new-tally-main/cloud/frontend/js/vouchers.js), [history.js](file:///c:/Users/Administrator/Downloads/new-tally-main/cloud/frontend/js/history.js), [auth.js](file:///c:/Users/Administrator/Downloads/new-tally-main/cloud/frontend/js/auth.js)

The current self-cleanup pattern (check `document.body.contains(body)` on next event) is lazy. Add immediate cleanup when the router replaces the view content. Two approaches:

**Option A:** The router could emit a `beforeRender` event that the current view listens to for teardown.

**Option B (simpler, surgical):** Remove the listener immediately after `main.innerHTML = htmlText` in [router.js L63](file:///c:/Users/Administrator/Downloads/new-tally-main/cloud/frontend/js/router.js#L63), by having each view return a cleanup function stored as `window._absViewCleanup`.

**Impact:** Releases closures holding old voucher data arrays, DOM references, and row state immediately on navigation, instead of on the next company-change event.

---

## User Review Required

> [!IMPORTANT]
> **Priority 4 — Touch throttling**: Option A (time-based throttle) is simpler but keeps the synchronous-session-in-async pattern. Option B (session reuse) is cleaner but requires more structural change to the WebSocket handler. Which approach do you prefer?

> [!IMPORTANT]
> **Priority 6 — View teardown**: Option A (router event) is more robust for future views but touches the router. Option B (`_absViewCleanup` pattern) is surgical and scoped to the three affected views. Which approach do you prefer?

## Verification Plan

### Automated Tests
- Not applicable — this codebase has no test suite, and memory leak verification requires runtime profiling.

### Manual Verification
1. **Before fix**: Deploy to a staging instance and run a load script that simulates 200+ cumulative user actions (upload → draft → push → delete cycles). Monitor RSS via `ps` or Railway's memory graph over 30 minutes. Note the monotonic growth.
2. **After fix**: Deploy the patched version to the same staging instance. Run the identical load script. Verify that RSS stabilizes (plateaus) within ~5 minutes of idle after the actions complete, and does not grow beyond the initial load-driven peak.
3. **OpenAI client**: Confirm only one `httpx` connection pool exists in the process by adding a temporary `log.info("OpenAI client id: %s", id(_openai_client))` — every AI call should log the same id.
4. **Post-lock dict**: After disconnecting a connector, verify `len(hub._post_locks)` decreases by one.
5. **Frontend timer**: Log out, wait 30 seconds, check browser DevTools Network tab — no `/connector/status` requests should fire after logout.
