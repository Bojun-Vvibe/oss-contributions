# QwenLM/qwen-code #3827 — refactor(core): unify retry delay policy

- **Head SHA reviewed:** `030a6b1d1370dde580b065dfe04f394bccd98705`
- **Size:** +476 / -145 across 5 files
- **Verdict:** merge-after-nits

## Summary

Refactor that consolidates the project's three formerly-distinct
retry delay implementations into a single `retryPolicy.ts` helper.
Behavior of the two callers (`rateLimit.ts`'s
`getRateLimitRetryDelayMs` and `retry.ts`'s `retryWithBackoff`) is
preserved by exposing a `retryAfterMode` knob (`'minimum'` for the
stream-side rate limiter, the persistent variant for `retryWithBackoff`).

Net code is +331 with two new test files
(`retry.test.ts:+107`, `retryPolicy.test.ts:+183`) and a sizeable
deletion in `rateLimit.ts` (-82) where the local Retry-After parsing
helpers move into the shared module.

## What I checked

- `packages/core/src/utils/rateLimit.ts:102-115` — old
  `getRateLimitRetryDelayMs` is now a thin shim that delegates to
  `getRetryDelayMs({ attempt, initialDelayMs, maxDelayMs,
  retryAfterMode: 'minimum', retryAfterMaxDelayMs: maxDelayMs, error })`.
  `'minimum'` mode preserves the original semantics — server's
  `Retry-After` is treated as a *floor* on top of exponential backoff,
  not a ceiling. Good.
- `rateLimit.ts:-187 to -270` (deletions) — local
  `getRetryAfterDelayMs`, `getHeaderValue`, `getResponseHeaderValue`,
  `hasHeaders`, `hasResponseHeaders` all removed. Confirmed via
  `retryPolicy.ts:+131` they're re-implemented in the shared module
  with the same shape (Headers `.get()` API + plain object iteration +
  `Date.parse` fallback). No behavior change visible in the diff.
- `retry.test.ts:+107` — adds a new `describe` block specifically
  exercising the `Retry-After handling in persistent mode` semantics
  (which is the path `retryWithBackoff` uses). The fact that this
  block exists and tests the *contract* the refactor is supposed to
  preserve is a strong signal.
- `retryPolicy.test.ts:+183` — covers the helper directly:
  exponential growth, jitter (assuming jitter is in the helper, as
  the PR body claims), header parsing edges (string seconds, HTTP
  date, malformed), and the `'minimum'` vs persistent
  `retryAfterMode` switch. Good standalone coverage.

## Concerns / nits

1. **Behavior preservation claim relies on test coverage.** The
   refactor is large, deletes ~145 lines, and the only protection
   against subtle drift is the new tests. I'd want to see the two
   pre-existing call sites' tests (the original
   `getRateLimitRetryDelayMs` tests, if any) reused unchanged to
   prove parity, not new tests written against the new API. Worth
   asking the author whether any pre-existing test was deleted or
   modified — `git log -p packages/core/src/utils/retry.test.ts` on
   the PR branch will tell.
2. **`retryAfterMode: 'minimum'` is an undocumented enum.** Add a
   JSDoc on `retryPolicy.ts` explaining what `'minimum'` vs the
   default mode mean operationally (floor vs replace), so future
   callers don't have to read the implementation.
3. **Jitter source.** The PR body promises jitter; confirm the helper
   uses a deterministic-seedable source so tests aren't flaky. The
   test file is +183 lines, presumably stubs `Math.random`, but worth
   a glance.
4. **Naming.** `retryAfterMaxDelayMs` is redundant with `maxDelayMs`
   in the only caller that sets them equal. Either drop it or
   document the case where they should diverge.

## Risk

Medium-low. Pure refactor, behavior-preserving by design, but the
surface area touched (retry/backoff is core to every API call) means
any subtle drift in jitter, header parsing, or precedence will show
up as flaky retries in production. The new tests look thorough.

## Recommendation

Merge after (a) confirming pre-existing rate-limit tests still pass
unchanged and (b) JSDoc-ing the `retryAfterMode` enum. Worth a
follow-up: collapse `retryAfterMaxDelayMs` if it's always equal to
`maxDelayMs` at every call site.
