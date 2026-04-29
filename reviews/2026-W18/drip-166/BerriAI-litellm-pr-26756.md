# BerriAI/litellm #26756 — fix(proxy): self-heal Prisma read paths + harden reconnect state machine

- **PR:** https://github.com/BerriAI/litellm/pull/26756
- **Head SHA:** `aa2ef4120098981cb6160d94347743e2e77ffb61`
- **Size:** ~+635 / −small across `db/exception_handler.py` (+135), `proxy/utils.py` (~+25/−15), and 380 lines of new/expanded tests

## Summary
Restores the 1.82.6 "transient `httpx.ReadError` self-heals via one reconnect-and-retry" behavior on `PrismaClient.get_generic_data` (regression in 1.83.x, issue #25143). Extracts the previously inlined retry pattern (formerly only in `auth_checks._fetch_key_object_from_db_with_reconnect`) into a canonical `call_with_db_reconnect_retry(prisma_client, coro_factory, *, reason, ...)` helper at `litellm/proxy/db/exception_handler.py:128-260`. Separately: fixes a state-machine bug in `_run_reconnect_cycle` where `self._engine_confirmed_dead = False` was clearing the flag *before* the heavy reconnect actually completed, so a heavy-reconnect failure would silently demote subsequent reconnects to the lightweight path forever.

## Specific observations

1. **The canonical helper at `db/exception_handler.py:152-260`** is the right shape:
   - Accepts `coro_factory: Callable[[], Awaitable[Any]]` (zero-arg, returns a fresh awaitable each call) — correctly avoids the `RuntimeError: cannot reuse already awaited coroutine` trap. Doc comment is explicit about this contract.
   - Bails fast if the exception is not a transport error (`PrismaDBExceptionHandler.is_database_transport_error`) — correct, because data-layer errors like `UniqueViolationError` mean the DB is reachable and reconnect would mask the real bug.
   - Bails fast if `prisma_client` lacks `attempt_db_reconnect` — guards against partial mocks and older clients.
   - **At-most-one retry by construction** (the second `await coro_factory()` is unguarded). Doc-comment-rationale: "repeated reconnect-loops are the watchdog's job, not this helper's." Correct.

2. **Defensive `_coerce_timeout` at `:135-141`** handles tests that hand a `MagicMock()` as `prisma_client._db_auth_reconnect_timeout_seconds`. Without this, `asyncio.wait_for(..., timeout=MagicMock())` would explode in test runs that mock the client. Good operational hygiene; the `bool` exclusion (`not isinstance(value, bool)`) prevents `True` (which is a `int` subclass) from becoming `1.0` silently — small but correct.

3. **The reconnect-attempt exception-chaining at `:243-251`** is the most thoughtful bit:
   ```python
   try:
       did_reconnect = await prisma_client.attempt_db_reconnect(...)
   except Exception as reconnect_exc:
       verbose_proxy_logger.warning(
           "DB reconnect attempt raised; preserving original transport error. "
           "reason=%s reconnect_error=%s", reason, reconnect_exc,
       )
       raise first_exc from reconnect_exc
   ```
   This preserves `first_exc` as the surfaced exception (so the `db_exceptions` alert correctly identifies the *original* transport problem) while chaining the reconnect failure as `__cause__` for debugging. The naive choice — letting `reconnect_exc` propagate — would have masked the real DB problem in alerts. This is exactly right.

4. **`get_generic_data` rewrite at `proxy/utils.py:2783-2820`** wraps each `find_first` branch in an inner `_do_query` factory and calls the helper with `reason=f"prisma_get_generic_data_{table_name}_lookup_failure"`. The per-`table_name` reason tag is correct — `_update_config_from_db` issues four concurrent reads, and telemetry needs to distinguish which one flapped. Good operational design.

5. **The state-machine fix at `proxy/utils.py:4204-4242`** is the one I'd call out the most:
   ```python
   -            self._engine_confirmed_dead = False
   ...
   ...
                await asyncio.wait_for(_do_heavy_reconnect(), timeout=effective_timeout)
   +            # Only clear the "dead engine" flag after the heavy reconnect
   +            # actually completed. ...
   +            self._engine_confirmed_dead = False
   ```
   Pre-fix: setting `self._engine_confirmed_dead = False` *before* awaiting the heavy reconnect meant that if `_do_heavy_reconnect()` raised (timeout, missing `DATABASE_URL`, recreate failure), the flag stayed false. Subsequent reconnect attempts would then take the lightweight branch (since `engine_alive_or_unknown == True`) instead of re-entering heavy reconnect. This bug class is "the cleanup ran before the work succeeded" — a single-line move with a multi-paragraph rationale, exactly the right amount of comment. The test file `test_exception_handler_reconnect_retry.py` (255 lines) plus the existing `test_prisma_self_heal.py` extension (+125 lines) lock this in.

6. **Logging discipline:** the helper emits exactly two `verbose_proxy_logger.warning` calls per failed attempt (one for "transport error, reconnecting", one for "reconnect itself raised"). No bare `print`, no logging in the success path. Correct — the success path should be silent so the warning rate equals the actual incident rate.

## Risks

- **`coro_factory` being awaitable-not-callable** is a correctness footgun the doc-comment calls out, but there's no runtime assertion (e.g., `inspect.iscoroutinefunction(coro_factory)`). A caller that hands a coroutine instead of a factory would only fail on the retry path — which by definition is rare. A `TypeError` at first invocation would be fail-loud. Cheap to add.
- **`is_database_transport_error` is the gate** for whether retry happens. Any new exception class that should be considered transient (e.g. `prisma.errors.PrismaClientConnectionError` if Prisma adds new ones) needs to be added there or the bug class returns silently. Worth a code-search pass to confirm coverage.
- **Concurrent `get_generic_data` callers** — if four reads (for `users`, `keys`, `config`, `spend`) all transient-fail simultaneously, all four will call `attempt_db_reconnect`. The reconnect lock should serialize them, but if `lock_timeout_seconds` is small (default 0.1s), three of the four will fail to acquire the lock and re-raise. That's the right behavior (one reconnect attempt per cycle) but worth verifying the four-concurrent-callers test exists.
- **`raise first_exc from reconnect_exc`** preserves the type for `is_database_transport_error` matching upstream, but consumers relying on `isinstance(e, type(reconnect_exc))` for branching would now see `first_exc`. Almost certainly fine; flagging because exception-chaining semantics across alerting integrations are easy to break.

## Suggestions

- **(Recommended)** Add a runtime check at the top of `call_with_db_reconnect_retry`: `if not callable(coro_factory): raise TypeError("coro_factory must be a zero-arg callable returning an awaitable")`. Trades zero perf for fail-loud-on-misuse.
- **(Recommended)** Add a four-concurrent-callers test (parametrize `_do_query` with all four `table_name`s, fail all four with `httpx.ReadError`, assert exactly one `attempt_db_reconnect` succeeds and the other three re-raise via the lock-timeout path). Pins the dedup contract.
- **(Optional)** Promote the helper from `litellm/proxy/db/exception_handler.py` to its own `litellm/proxy/db/retry.py` once the second/third caller migrates — keeps `exception_handler.py` focused on the exception-classification surface.
- **(Optional)** The state-machine fix (`_engine_confirmed_dead`) is unrelated to the helper extraction; release notes should split them so users debugging "stuck on lightweight reconnect" know to look for this PR.

## Verdict: `merge-after-nits`

Correct restoration of self-heal behavior, correct extraction to a single canonical helper, correct exception-chaining, and a load-bearing state-machine fix that prevents the heavy-reconnect path from being permanently demoted. The nits are a runtime fail-loud check on the factory contract and one missing concurrent-callers test.

## What I learned

When restoring a regressed behavior, the fix shape is "extract the inline pattern that already worked into a helper, then wire it back into the regressed call site." This PR does exactly that — the pattern existed in `auth_checks._fetch_key_object_from_db_with_reconnect`, was lost from `get_generic_data` in the 1.83 refactor, and gets restored as a shared helper instead of as an inlined re-paste. Net: one canonical implementation rather than three drifting copies. The bonus state-machine fix is the kind of bug that only shows up under a specific failure-of-failure ordering — set-the-flag-before-the-work pattern. Worth scanning the codebase for similar shapes.
