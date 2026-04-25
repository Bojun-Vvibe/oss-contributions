# cline/cline #10384 — fix: cap retry-after delay to prevent silent multi-hour hangs

- **Repo**: cline/cline
- **PR**: [#10384](https://github.com/cline/cline/pull/10384)
- **Head SHA**: `65b1c5e2993f0cc7b8e3d1cc9b06300d57120b3e`
- **Author**: NgoQuocViet2001
- **State**: OPEN (+68 / -7)
- **Verdict**: `merge-after-nits`

## Context

`withRetry` decorator honors HTTP `Retry-After` headers. If a
provider returns a hostile or accidental
`Retry-After: 10800` (3 hours) on a 429, the decorator silently
sleeps the user's session for that whole window. Fix adds a
`maxRetryAfter` ceiling (default 60 s) — past that, throw
immediately rather than hang.

## Design

`src/core/api/retry.ts`:

1. **Lines 4-15**: New optional `maxRetryAfter?: number` on
   `RetryOptions`, default 60_000 (60 s) in
   `DEFAULT_OPTIONS`.

2. **Line 32**: Destructured into `withRetry`'s scope.

3. **Lines 58-72**: After computing `delay` from `Retry-After`
   (either delta-seconds or Unix-timestamp form), check
   `if (delay > maxRetryAfter) throw error`. The original
   error propagates, so callers see the real 429 instead of a
   silent multi-hour stall.

4. **Lines 18, 32, 61**: Drive-by hygiene —
   `parseInt → Number.parseInt`, `status: number = 429 →
   status = 429`. Lint-prefer-static rules; harmless.

Tests at `src/core/api/retry.test.ts`:
- **Lines 113-145**: Existing Unix-timestamp test rewritten to
  stub `Date.now` instead of using a `2010` literal date. Math
  is now `(fakeNow/1000 + 1) * 1000 - fakeNow = 1000ms` —
  asserted as the actual `setTimeout` delay. Cleaner; was
  previously brittle against system clock.
- **Lines 181-208**: New
  `should throw immediately when retry-after exceeds
  maxRetryAfter` — sets `retry-after: 10800` (3 hours) with
  `maxRetryAfter: 60_000`, asserts the original error throws
  and `callCount === 1` (no retry attempted).
- **Lines 210-234**: Companion happy-path test —
  `retry-after: 0.01` (10 ms), `maxRetryAfter: 60_000`, asserts
  the retry does happen.

## Risks

- **Default 60 s may be too short for some legitimate cases.**
  Anthropic and OpenAI occasionally return `Retry-After`
  values in the 60-300 s range during incidents; with the new
  default, those will now throw instead of waiting. The fix is
  per-call (`@withRetry({ maxRetryAfter: 600_000 })`), but the
  default should be picked deliberately. 60 s feels conservative
  for an interactive coding agent (good — fail fast, surface to
  user) but may break a long-running agent loop. Worth a PR-body
  note about the chosen default.
- **Throws the original error, not a wrapped "exceeded
  maxRetryAfter" error.** From the caller's perspective, a
  429 with a 3-hour delay is now indistinguishable from a 429
  with no delay header (both throw immediately). Consider
  attaching `error.retryAfterTooLong = true` or wrapping in
  `RetriableError` with a distinct message so the UI can render
  "rate limited until X" instead of a generic 429 message.
- **`delay > maxRetryAfter`** doesn't bound *negative* delays —
  if the Unix-timestamp branch (lines 60-62) computes a stale
  past-time, `delay` is negative and `setTimeout` runs
  immediately. That's fine behaviour but worth a `Math.max(0,
  delay)` for clarity.
- **No coverage for the boundary case** `delay === maxRetryAfter`
  (currently `>`, so equal-to passes through). Negligible.
- **`maxDelay` (10 s) and `maxRetryAfter` (60 s) are now two
  ceilings with overlapping semantics.** A `Retry-After: 30`
  header gives `delay = 30_000`, which is *over* `maxDelay
  (10_000)` but *under* `maxRetryAfter (60_000)`. The code
  honors the 30 s header (correctly — `Retry-After` should
  override exponential backoff caps) but the relationship
  between the two ceilings deserves a doc comment so future
  readers don't conflate them.

## Suggestions

- Add a one-line PR-body note justifying the 60 s default.
- Wrap the thrown error so callers can distinguish
  "exceeded maxRetryAfter" from a generic 429.
- Add `Math.max(0, delay)` defensive clamp for stale Unix
  timestamps.
- Add a doc comment to `RetryOptions` explaining the
  `maxDelay` vs. `maxRetryAfter` distinction.

## What I learned

`Retry-After` is one of those HTTP headers that's
spec-compliant to set arbitrarily large, but real consumers
need a sanity ceiling. A multi-hour silent hang in an
interactive agent is much worse UX than a clear "we gave up
after 60 s, please try again" — the user can decide whether
to wait. The pattern (configurable ceiling, default low,
opt-in higher per-call) is the right shape for any retry
library that honors server-supplied delays.
