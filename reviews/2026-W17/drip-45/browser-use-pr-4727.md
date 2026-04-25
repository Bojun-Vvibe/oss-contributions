# browser-use/browser-use #4727 — fix(browser): replace deprecated asyncio.get_event_loop() with get_running_loop()

- **PR:** https://github.com/browser-use/browser-use/pull/4727
- **Head SHA:** `423b13eb73bc38de9c597f0c6ea2a117fa7ec65f`
- **Files changed:** 5 — `session.py`, `session_manager.py`, `watchdogs/downloads_watchdog.py`, `watchdogs/local_browser_watchdog.py`, `watchdogs/recording_watchdog.py`

## Summary

Mechanical replacement of `asyncio.get_event_loop()` with `asyncio.get_running_loop()` across browser-session and watchdog code. Targets Python 3.12+ deprecation — `get_event_loop()` without a running loop emits a `DeprecationWarning` in 3.12 and is slated to raise in a future version.

## Line-level call-outs

- `browser_use/browser/session.py:982,1000,1006,1011,1031,1044,1054` — all 7 swaps are inside `async def _navigate_and_wait(...)`. Since the function is `async`, there is *always* a running loop at these points → `get_running_loop()` is a strict win (no behaviour change, plus removes the deprecation noise). Correct.
- `browser_use/browser/session_manager.py:340,342` — inside `async def ensure_valid_focus(...)`. Same reasoning, safe.
- `browser_use/browser/session_manager.py:889` — **flag**: this is inside a `def on_lifecycle_event(event, session_id=None):` callback at line 880. If the callback is invoked from a synchronous CDP-event dispatch (i.e. not from inside an awaiting coroutine), `get_running_loop()` will raise `RuntimeError("no running event loop")`. Need to confirm this callback is only fired from within the asyncio event-loop thread (e.g. via `loop.call_soon_threadsafe` or as a coroutine). If it can be invoked from a CDP worker thread or a synchronous PyObject callback, this changes a deprecation warning into a hard crash. **This is the one risky swap in the PR** — please verify with a test that exercises the lifecycle event path.
- `browser_use/browser/watchdogs/downloads_watchdog.py:884,886` — inside `async def _handle_cdp_download(...)`. Safe.
- `browser_use/browser/watchdogs/local_browser_watchdog.py:412,414` — inside `async def _wait_for_cdp_url(...)`. Safe.
- `browser_use/browser/watchdogs/recording_watchdog.py:116` — inside `async def stop_recording(...)`. Safe. Note: `loop.run_in_executor` requires a running loop, so this was already implicitly relying on one.
- The `# noqa: ASYNC110` on `local_browser_watchdog.py:413` and `downloads_watchdog.py:885` is preserved — good, those polling loops still trip ASYNC110 regardless of which `get_*_loop` is used.

## Verdict

**merge-after-nits**

## Rationale

10 of 11 swaps are in `async def` contexts and are unambiguously correct. The **single** swap in `session_manager.py:889` is inside a CDP event callback whose calling context is non-obvious from the diff alone — if that callback ever fires from a non-asyncio thread, this PR turns a future-warning into a hard `RuntimeError`. Author should either (a) confirm the callback is always invoked from the loop thread (and ideally add a comment saying so), or (b) wrap that one site in `try: get_running_loop() except RuntimeError: get_event_loop_policy().get_event_loop()` for safety. Everything else is mechanical and safe.
