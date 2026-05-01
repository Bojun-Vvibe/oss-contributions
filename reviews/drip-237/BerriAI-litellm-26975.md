# BerriAI/litellm#26975 — fix: implement sync service failure hook

- **PR**: https://github.com/BerriAI/litellm/pull/26975
- **Head SHA**: `e8e0ed2927a2119722875aaa17bc6be68add2aaa`
- **Size**: +67 / -3, 2 files (`litellm/_service_logger.py`, `tests/test_litellm/test_service_logger.py`)
- **Verdict**: **request-changes**

## Context

`ServiceLogging.service_failure_hook` (the *sync* counterpart to `async_service_failure_hook`) was a no-op stub with a "TODO: V0 is focused on async monitoring" comment. The PR fills it in by trying to hand off to the async hook via three branches: (1) loop is running → `loop.create_task(...)`, (2) loop exists but not running → `loop.run_until_complete(...)`, (3) `RuntimeError` → `asyncio.run(...)`.

## What's right

- **The mock-testing counter still increments** at `_service_logger.py:104-105`, preserving the existing observability hook.
- **One regression test** at `tests/test_litellm/test_service_logger.py:107-126` confirms the running-loop branch schedules the async hook with the right kwargs.

## What's wrong

- **`asyncio.get_event_loop()` is the wrong API in modern Python.** At `_service_logger.py:111`, this is called inside a sync function. On Python 3.12+, calling `get_event_loop()` from a thread that has no running loop emits a `DeprecationWarning` and on 3.14 it will raise. The correct discipline is `try: loop = asyncio.get_running_loop()` (which never creates a loop and raises `RuntimeError` cleanly when there isn't one) — *then* fall back to `asyncio.run(...)`.
- **The "loop exists but not running" branch is reachable from a running event loop's thread and will deadlock.** At `:118-126`: `loop.run_until_complete(...)` called from a thread that *owns* the running loop raises `RuntimeError: This event loop is already running`. The branch above (`loop.is_running()` → `create_task`) is supposed to catch that case, but `get_event_loop()` in a worker thread (e.g. a sync background-task executor) may return *some other thread's* loop that "exists but isn't running in this thread" — `run_until_complete` then blocks the worker thread forever waiting on the cross-thread loop.
- **`asyncio.run(...)` at `:131-138` will silently fail on the second sync call from the same thread.** `asyncio.run` requires no running loop AND tears down the loop it creates. If the caller is itself inside an `asyncio.run` block (common in proxy code), this branch is unreachable; if the caller is a thread that has previously called `asyncio.run`, the second call works but with the cost of a fresh loop spin-up (~ms-scale latency on a *failure path that's supposed to be fast*).
- **Fire-and-forget `loop.create_task` swallows exceptions.** At `:113-117`, the returned task is never awaited and has no `add_done_callback` for error logging. Any failure in `async_service_failure_hook` (the very thing we're trying to monitor) becomes a "Task exception was never retrieved" warning at GC time and otherwise vanishes. The whole point of a service-failure hook is to *not* lose failures.
- **The test at `:107-126` only covers the running-loop branch.** The two other branches (`loop exists but not running` and `RuntimeError → asyncio.run`) have zero test coverage despite being the load-bearing differentiation in the new code. The branch most likely to deadlock in production has zero test.
- **No timeout on the async hook dispatch.** A misbehaving callback that hangs in `async_service_failure_hook` will block `run_until_complete`/`asyncio.run` indefinitely on the sync caller's thread.

## Suggested approach

Sync-from-async dispatch in modern Python is best done with one of two well-known patterns:
1. **`asyncio.run_coroutine_threadsafe(coro, loop)`** when there's a known background loop — returns a `concurrent.futures.Future` that can be `.result(timeout=...)`-bounded or fire-and-forget with `.add_done_callback(log_exc)`.
2. **`anyio.from_thread.run`** if anyio is acceptable as a dependency — handles the cross-thread/cross-runtime cases correctly without the deprecation surface.

Either way, the retry/fire-and-forget choice should be explicit and the error path should at minimum log via `verbose_logger.exception` so a hung or failing callback is visible.

## Verdict

**request-changes.** The intent is right (the TODO has been a footgun for callers who assumed sync calls were monitored) but the chosen implementation introduces a deprecation-bound API call, a cross-thread deadlock surface on the `run_until_complete` branch, silent swallowing of exceptions on the `create_task` branch, and zero coverage for the two branches most likely to misbehave. Worth resubmitting with `get_running_loop` + `run_coroutine_threadsafe` (or `asyncio.run` only when the caller is provably loop-less) and tests for all three branches including the no-loop and existing-but-not-running cases.
