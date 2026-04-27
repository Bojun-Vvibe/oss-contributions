# BerriAI/litellm #26578 — fix(proxy): avoid blocking event loop by killing engine instead of disconnecting

- **Repo**: BerriAI/litellm
- **PR**: #26578
- **Author**: dschulmeist
- **Head SHA**: d7385097cf1299be3fac95f6c2a7d5426733f37e
- **Base**: main
- **Size**: +79 / −125 across four files:
  `litellm/proxy/db/prisma_client.py` (+15/-12),
  `litellm/proxy/utils.py` (+6/-8),
  `tests/test_litellm/proxy/db/test_prisma_client.py` (+14/-68),
  `tests/test_litellm/proxy/db/test_prisma_self_heal.py` (+44/-37).

Re-opening of the previously closed #26329 against the new OSS
staging branch. Fixes #26191.

## What it changes

The Prisma reconnect paths in the LiteLLM proxy used to call
`await self.db.disconnect()` (or `await
self._original_prisma.disconnect()`) before re-opening a fresh
connection. Under prisma-client-py, `disconnect()` ultimately
invokes a synchronous `process.wait()` on the query-engine
subprocess. When the database is unreachable the engine doesn't
exit promptly, so `process.wait()` blocks the asyncio event loop
for 30-120 seconds — long enough for the `/health/liveliness` probe
to fail and the orchestrator to restart the pod, defeating the
self-heal loop entirely.

This PR replaces the `disconnect()` path with a direct
`SIGTERM`-then-`SIGKILL` of the query-engine PID:

1. `prisma_client.py:218-235`: `recreate_prisma_client` no longer
   calls `disconnect()`. It captures the old engine PID via
   `self._get_engine_pid()` and unconditionally calls
   `await self._kill_engine_process(old_engine_pid)` before
   instantiating a fresh `Prisma(...)` and `connect()`-ing it.
2. `prisma_client.py:62-78`: `_kill_engine_process` is reframed —
   the docstring at `:62-69` updates from "Force-kill an *orphaned*
   engine subprocess to prevent DB connection pool leaks" to "the
   reconnect paths use this in place of `disconnect()`, whose
   synchronous `process.wait()` blocks the asyncio event loop for
   30-120s on DB outages (#26191)". The log line at `:75` changes
   from "Sent SIGTERM to *orphaned* prisma-query-engine PID %s
   after failed disconnect" to "Sent SIGTERM to prisma-query-engine
   PID %s during client reconnect". Same primitive, narrowed and
   relabeled to its new role.
3. `utils.py:4134-4145`: the inner `_do_direct_reconnect` of the
   "heavy reconnect" path gets the same treatment. The previous
   `try: await self.db.disconnect() except: kill` is replaced by
   an unconditional `kill` followed by `await self.db.connect()`
   and `await self.db.query_raw("SELECT 1")`.

The test changes pin the new contract:

- `test_prisma_client.py:46-83` deletes the previous
  `test_recreate_prisma_client_successful_disconnect` (no longer
  applicable) and the
  `test_recreate_prisma_client_skips_kill_on_successful_disconnect`
  (the contract is inverted now). The renamed
  `test_recreate_prisma_client_kills_old_engine_without_disconnect`
  uses `mock_prisma.disconnect.side_effect = AssertionError(
  "disconnect would block the event loop")` so any regression that
  re-introduces a `disconnect()` call fails loudly. It then
  asserts both `mock_kill.assert_any_call(12345, signal.SIGTERM)`
  and `mock_prisma.disconnect.assert_not_called()` — the second
  assertion is the regression-prevention bar.

## Strengths

- The fix targets the root cause precisely: a sync `process.wait()`
  inside `disconnect()` is fundamentally incompatible with an
  asyncio event loop that has a 5-10s liveness budget. No amount
  of `try`/`except` around `disconnect()` recovers the lost time;
  removing the call entirely is the only correct fix.
- The new test idiom (`mock_prisma.disconnect.side_effect =
  AssertionError("disconnect would block the event loop")` at
  `test_prisma_client.py:60-62`) is excellent regression
  prevention. A future refactor that "helpfully" adds a
  best-effort `disconnect()` back will fail this test on the
  first run, with an assertion message that explains exactly why.
  Pin the contract by failing fast on the forbidden call.
- Both reconnect call sites get the same treatment, in lockstep
  (`prisma_client.py:222-225` and `utils.py:4137-4143`). Easy to
  miss one of two — getting both is the right discipline for an
  event-loop-blocking class of bug.
- The docstring rewrite at `prisma_client.py:62-69` and `:218-227`
  doesn't just describe the change — it includes the issue number
  (#26191) and the latency window (30-120s) so the next maintainer
  sees the reasoning without git-blaming.
- The deleted tests
  (`test_recreate_prisma_client_successful_disconnect` and
  `_skips_kill_on_successful_disconnect`) are the right ones to
  delete — they were testing a contract this PR explicitly
  reverses. Keeping them as expected-fail or `xfail` would be a
  trap for the next reader.
- The `_do_heavy_reconnect` path (`utils.py:4115-4133`, unchanged)
  remains as the "let prisma-client-py do its thing" fallback;
  the direct path at `_do_direct_reconnect` is the new fast-path.
  The two-tier structure is preserved.

## Risks / nits

- Killing the engine subprocess with `SIGTERM`/`SIGKILL` will
  abort any in-flight queries on that engine. The previous
  `disconnect()` path at least *attempted* graceful drain. For
  the DB-unreachable case (the issue this PR targets) there's
  nothing to drain, so the change is correct. But for the
  *new-DB-URL-rotation* case (`recreate_prisma_client`'s other
  caller, key rotation / IAM token refresh), in-flight queries
  against the old engine will now be killed mid-flight where
  before they'd have a brief drain window. That's almost
  certainly fine — the caller usually wants the new credential
  to take effect immediately — but worth a sentence in release
  notes for operators who'd been relying on graceful drain.
- `_kill_engine_process` (`prisma_client.py:62-90`) sends `SIGTERM`
  with a 0.5s wait then `SIGKILL`. On a slow CI host or under heavy
  IO contention, 0.5s is occasionally too short for graceful exit;
  the `SIGKILL` then orphans any temp files the engine had open.
  Not a regression (this code path existed before), but the new
  hot-path positioning makes it more visible. Worth measuring
  whether the wait is sufficient under realistic load.
- `_get_engine_pid()` returning `0` (or negative) is handled at
  `:71-72` (early return). But there's no log when the engine PID
  is unavailable — silent no-op. Worth a debug log so the
  reconnect path is observable end-to-end.
- The inverted contract test
  (`test_recreate_prisma_client_kills_old_engine_without_disconnect`)
  uses `AssertionError` as the `side_effect`, but `AsyncMock`
  side-effects are raised on `await`. The test relies on
  `recreate_prisma_client` *not* awaiting `disconnect()` — if
  someone added a synchronous `mock_prisma.disconnect()` call
  (without `await`), the side_effect wouldn't fire on the
  unawaited coroutine warning. Belt-and-braces would be to also
  assert `mock_prisma.disconnect.call_count == 0` (which the test
  already does via `assert_not_called()`).
- The change in `utils.py:4134-4143` removes the `try`/`except`
  around the kill call. If `_kill_engine_process` itself raises
  (e.g., `os.kill` permission denied), the reconnect now
  propagates the error rather than swallowing it and proceeding.
  That's arguably more correct (don't silently leak a stale
  engine), but it's a behavior change worth documenting.

## Suggestions

- Add a release-notes line for operators: "Engine subprocesses
  are now SIGTERMed during reconnect rather than gracefully
  disconnected; in-flight queries against the old engine will be
  aborted. This trade-off is required to keep liveness probes
  responsive during DB outages."
- Add a debug-level log when `_get_engine_pid()` returns
  `<= 0` so an operator can distinguish "killed the engine"
  from "no engine to kill" in logs.
- Consider parameterizing the SIGTERM-to-SIGKILL wait (currently
  hardcoded at `_kill_engine_process`) so noisy environments can
  extend it without forking the proxy.
- A follow-up to add an integration test that actually exercises
  the DB-unreachable path (e.g., `iptables -A OUTPUT -p tcp
  --dport 5432 -j DROP` in a container) and asserts the liveness
  probe responds within `<5s` would close the loop on the
  reported issue.

## Verdict

`merge-as-is` — the fix is targeted, correct, and the
regression-prevention test idiom (`disconnect.side_effect =
AssertionError`) is the right bar for an event-loop-blocking
class of bug. Suggestions are follow-ups, not blockers. The
release-notes mention is the only thing I'd ask for explicitly,
but it doesn't have to be in this PR.
