# PR #10384 — fix: cap retry-after delay to prevent silent multi-hour hangs

- **Repo**: cline/cline
- **PR**: #10384
- **Head SHA**: `65b1c5e2993f0cc7b8e3d1cc9b06300d57120b3e`
- **Author**: NgoQuocViet2001
- **Size**: +68 / -7 across 2 files
- **Verdict**: **merge-as-is**

## Summary

The `withRetry` decorator honors the `retry-after` header to
schedule the next retry, supporting both delta-seconds and
unix-timestamp formats. There was no upper bound. A misbehaving
provider returning `retry-after: 10800` (3 hours) — or worse, a
unix timestamp far in the future — would leave the request
silently sleeping for hours without any feedback to the user.
This PR adds a `maxRetryAfter` option (default 60_000 ms): if
the computed delay exceeds it, the original error is rethrown
immediately instead of waiting.

## Specific changes

- `src/core/api/retry.ts:7` — `RetryOptions.maxRetryAfter?:
  number` added to the public type.
- `retry.ts:14` — `DEFAULT_OPTIONS.maxRetryAfter: 60_000`. One
  minute is a reasonable ceiling for an agent loop; users hitting
  longer rate-limit windows will see the actual error rather
  than a hung process.
- `retry.ts:30-32` — destructures the new option from the merged
  options.
- `retry.ts:58` — `parseInt` → `Number.parseInt`. Not the fix,
  just lint-cleanup that came along for the ride.
- `retry.ts:69-71` — the actual fix:
  ```ts
  if (delay > maxRetryAfter) {
      throw error
  }
  ```
  Throws *the original error*, not a custom "delay too long"
  wrapper. Good — preserves the upstream's status code and
  metadata for the caller's error handler.
- `retry.test.ts:115-117` — existing unix-timestamp test
  rewritten to use a `sinon.stub(Date, "now")` so the assertion
  is deterministic instead of dependent on the wall clock at the
  fixed-date.
- `retry.test.ts:181-207` — new test:
  `"should throw immediately when retry-after exceeds maxRetryAfter"`
  — sends `retry-after: 10800` (3 hours), expects exactly 1 call
  (no retry), error message preserved.
- `retry.test.ts:209-232` — symmetric test:
  `"should still retry when retry-after is within maxRetryAfter"`
  — sends `retry-after: 0.01` (10ms), expects 2 calls and success.

## Risks

Low. Two minor things worth noting:

1. **Behavior change for existing callers**: anyone relying on
   the previous unbounded behavior will start failing fast on
   long retry-after windows. That's almost certainly a *fix*
   for them, not a regression — but worth a release-note line.
2. The default 60s ceiling may be aggressive for batch workloads
   that *want* to wait out a long rate-limit window. Since
   `maxRetryAfter` is settable per-call via `@withRetry({
   maxRetryAfter: 600_000 })`, the escape hatch is fine.
3. The `delay > maxRetryAfter` check is on the *computed* delay,
   which means the unix-timestamp branch (delay = retryValue *
   1000 - Date.now()) is also bounded. Good — that's where the
   worst hangs come from.

## Verdict

`merge-as-is` — small, focused, well-tested fix for a real
foot-gun. The default value is debatable but the option is
overridable.

## What I learned

"Honor `retry-after` literally" is one of those defaults that
looks polite to the upstream and turns out to be hostile to the
user. Any sleep-based wait derived from network input needs a
ceiling; the network input is the *adversary* in the threat
model, not just the source of truth. Bonus pattern: when the
computed delay exceeds the ceiling, throwing the original error
(not a synthetic "too long" error) preserves the caller's
existing error-handling code path.
