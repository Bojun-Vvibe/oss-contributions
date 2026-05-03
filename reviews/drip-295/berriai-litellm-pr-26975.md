# BerriAI/litellm PR #26975 — fix: implement sync service failure hook

- **Repo:** BerriAI/litellm
- **PR:** #26975
- **Head SHA:** `e8e0ed2927a2119722875aaa17bc6be68add2aaa`
- **Author:** (see PR)
- **Title:** fix: implement sync service failure hook
- **Diff size:** +67 / -3 across 2 files
- **Drip:** drip-295

## Files changed

- `litellm/_service_logger.py` (+41/-2) — replaces the `[TODO] Not implemented for sync calls yet` stub of `service_failure_hook(service, duration, error, call_type)` with a three-branch event-loop probe that delegates to the existing `async_service_failure_hook`.
- `tests/test_litellm/test_service_logger.py` (+26/-1) — adds `test_service_failure_hook_should_schedule_async_failure_hook` asserting that the running-loop branch schedules a task and the underlying async hook gets awaited with the correct kwargs.

## Specific observations

- `_service_logger.py:104-110` — `loop = asyncio.get_event_loop()` is **deprecated when there is no current loop** (Python 3.10+) and emits a `DeprecationWarning`; in Python 3.12 it raises `DeprecationWarning` with stronger language and in some configurations raises outright. The intended modern API is `asyncio.get_running_loop()` which raises `RuntimeError` cleanly when no loop is running. The current code essentially relies on the deprecated implicit-creation behavior and then catches `RuntimeError` from a different code path. Recommend: `try: loop = asyncio.get_running_loop(); loop.create_task(...) except RuntimeError: asyncio.run(...)`. That's also simpler — the middle branch (`loop.run_until_complete`) is dead code in any realistic deployment because `get_event_loop()` won't return a non-running loop in Python 3.10+ from a sync context.
- `_service_logger.py:111-118` — `loop.create_task(self.async_service_failure_hook(...))` returns a `Task` that is **not retained anywhere**. Per Python docs, "save a reference to the result of [create_task], to avoid a task disappearing mid-execution. The event loop only keeps weak references to tasks." If the surrounding code returns and the task is GC'd before the hook completes, the hook is silently dropped. Stash the task on `self` (eg. `self._pending_failure_hooks: set[asyncio.Task] = set()`; `t = loop.create_task(...); self._pending_failure_hooks.add(t); t.add_done_callback(self._pending_failure_hooks.discard)`).
- `_service_logger.py:128-136` — the `RuntimeError` fallback runs `asyncio.run(self.async_service_failure_hook(...))` from a sync context. `asyncio.run` creates a fresh loop, runs the coro to completion, and tears the loop down. If `service_failure_hook` is called frequently from a sync context (eg. a request loop with no async runtime), this creates and destroys a loop per call — **expensive**. Worse, if any code path elsewhere has a current loop reference, this fallback will fight with it. Consider lazily creating a single dedicated background loop on a thread for the sync-only case.
- `_service_logger.py:104-110` — the `loop.is_running()` middle ground (`loop.run_until_complete(...)`) is dangerous: calling `run_until_complete` from inside a callback that itself runs on a non-running-but-existing loop is non-trivial to reach in practice, and if reached can deadlock. Drop this branch.
- `_service_logger.py:91-93` — the docstring change ("Handles both sync and async monitoring by checking for existing event loop.") is accurate but loses the historical "[TODO] V0 is focused on async monitoring (used by proxy)" context. If any caller still expects a no-op (eg. a proxy code path that intentionally uses sync to avoid logging overhead), this is now a behavioral change. Worth grepping `service_failure_hook(` callers in the codebase to confirm none relied on the prior no-op semantics.
- `tests/test_litellm/test_service_logger.py:78-103` — only the running-loop branch is tested (the test is `@pytest.mark.asyncio` so a loop is always running). The `RuntimeError` fallback (no loop) and the `loop.is_running() == False` branch have no coverage. At minimum add a test that runs `service_failure_hook` from a fresh thread without an event loop, asserts no exception, and asserts the underlying hook was invoked. The dropped-task GC behavior also wants a test (force GC after the call, await `mock_hook.assert_called`).
- `await asyncio.sleep(0)` at line 95 is the right idiom to yield to the scheduled task. Good.
- The test asserts `mock_hook.assert_awaited_once()` and the kwargs round-trip — solid for what it covers.

## Verdict: `merge-after-nits`

The intent is right (replace a known no-op stub with real behavior) and the test is well-shaped, but the loop-detection is a textbook anti-pattern: (1) replace `get_event_loop()` with `get_running_loop()` and drop the `loop.is_running()` middle branch — it's effectively dead code on Python 3.10+; (2) retain a strong reference to the scheduled task to prevent silent GC drops; (3) document or cap the `asyncio.run` per-call cost in the no-loop fallback. Add tests for the no-loop fallback and the GC-survives case before merge.
