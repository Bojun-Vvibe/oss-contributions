# BerriAI/litellm PR #26434 — fix shared health check polling

Link: https://github.com/BerriAI/litellm/pull/26434
State: OPEN · Author: noahnistler · +180 / −25 · Files: 2

## Summary
`SharedHealthCheckManager.perform_shared_health_check()` is the multi-pod coordinator that
elects one pod to run the health check and lets the others read cached results. The
non-leader branch was waiting exactly 2 seconds before falling back to a redundant local
health check. Real health checks against multi-model lists routinely take longer than 2s, so
the cache was almost never ready in time and every non-leader pod did its own redundant
work — defeating the purpose of the shared coordination. The PR replaces the single
`asyncio.sleep(2)` with a polling loop: poll every 5s for cached results, up to `lock_ttl`
(default 60s), exit early if the lock disappears without a cache write (crash recovery),
defensively swallow Redis errors during polling, skip polling entirely when `redis_cache is
None`, and fall back to local health check only after exhausting the wait. Adds three new
tests covering polling-finds-cache, polling-exhausts, orphaned-lock-early-exit, and
redis-error-mid-poll.

## Strengths
- Diagnosis is exact: a hard-coded 2s wait for an operation that takes longer is the textbook
  shape of a silently-degraded coordinator.
- The early-exit on lock disappearance is the right defense against a leader pod crashing
  mid-check; without it, all non-leaders would idle for the full `lock_ttl` (60s default)
  before realizing they should run their own.
- The `redis_cache is None` short-circuit prevents introducing a 60s regression for
  single-pod deployments that have no Redis configured.
- Test coverage now includes the failure modes that previously had no coverage (orphaned
  lock, Redis error mid-poll).
- Backwards-compatible signature, so this can land as a point release.

## Concerns / Risks
- **5-second poll interval is hard-coded.** For health checks that complete in 3–4 seconds
  (a small model list), non-leaders will still wait at least 5s before reading the cache,
  adding a near-100% latency overhead to the first request after the check window opens.
  An exponential-backoff or shorter initial poll (1s, then 5s) would let small fleets reap
  the speedup without changing behavior for large fleets.
- **Edge case: `lock_ttl` not divisible by `poll_interval`.** With `lock_ttl=10` and
  `poll_interval=5`, you get exactly 2 polls. With `lock_ttl=12`, you still get 2 polls and
  exit at `elapsed=10`, leaving 2 seconds on the lock before the local fallback. Acceptable,
  but worth a comment that elapsed accounting is poll-quantized.
- **Race: lock check after cache check.** Sequence within one iteration is `sleep` →
  `get_cached_results` → `check_lock_owner`. If the leader writes the cache *between* the
  cache check and the lock check, the next iteration will find the cache, but the current
  iteration also sees the lock as released and breaks the loop early — falling through to
  local fallback in the same call where the cache had just been written. The window is
  small (microseconds) but the cost is real (a redundant health check). Reordering to check
  cache once more after detecting lock release would close it.
- **Defensive `try/except` around `async_get_cache`** swallows the exception with no log,
  which makes a flaky Redis silently look like "still polling". A `verbose_proxy_logger.debug`
  would help operators correlate "polling forever" with Redis errors.
- **`elapsed` tracking is approximate** — `await asyncio.sleep(5)` may sleep more than 5s
  under load, but the loop accounting credits exactly 5s per iteration. Under sustained
  scheduler pressure the loop could run longer than `lock_ttl` thinks. Minor.
- **Test for the "5s poll interval" magic number** would help: changing it to 4s shouldn't
  break tests beyond the call-count assertion. Currently a refactor would break the brittle
  `mock_sleep.assert_called_with(5)` check.

## Suggested follow-ups
1. Make `poll_interval` configurable (env var or constructor arg) and add a shorter initial
   poll for the small-fleet case.
2. Re-check the cache after detecting lock release, before falling through to local check.
3. Log the swallowed Redis exception at debug level so flaky-Redis incidents are diagnosable.
