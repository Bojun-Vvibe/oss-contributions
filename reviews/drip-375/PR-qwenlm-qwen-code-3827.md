# QwenLM/qwen-code #3827 — refactor(core): unify retry delay policy

- **PR:** QwenLM/qwen-code#3827
- **Head SHA:** `030a6b1d1370dde580b065dfe04f394bccd98705`
- **Files:** 5 changed (+476 / -145) — net +331, but most of the additions are tests for the new module

## What changed

- New `packages/core/src/utils/retryPolicy.ts` (+131) is the single chokepoint for "given an attempt number and an error, how long should I wait?". It exposes `getRetryDelayMs({ attempt, initialDelayMs, maxDelayMs, jitterRatio?, retryAfterMode?, retryAfterMaxDelayMs?, error? })` plus the previously-private `getRetryAfterDelayMs` parser. The `retryAfterMode` switch is the design crux: `'minimum'` returns `max(exponential, retryAfter)` (used by stream-content retry where the server's hint is treated as a floor), while `'prefer'` returns `retryAfter` capped at a separate ceiling (used by interactive HTTP where the server's number wins outright). Companion test `retryPolicy.test.ts` (+183) pins both modes at multiple attempt counts.
- `packages/core/src/utils/retry.ts:189-196, 234-308` — the persistent-mode and normal-mode retry branches both delegate to `getRetryDelayMs(...)` instead of inlining `Math.pow(2, ...)` plus jitter plus min/max gymnastics. The local `getRetryAfterDelayMs` (lines 320-355 in the old file, deleted in this PR) is gone — the retry module now imports it from `retryPolicy.ts`. New constant `INTERACTIVE_RETRY_AFTER_CAP_MS = 5 * 60 * 1000` at `retry.ts:20` is the cap applied to interactive mode's `prefer`-mode call (caps a hostile server's `Retry-After: 86400` to 5 minutes for interactive sessions).
- `packages/core/src/utils/rateLimit.ts:105-113` — `getRateLimitRetryDelayMs` collapses from a 10-line bespoke implementation into a single `getRetryDelayMs({ ..., retryAfterMode: 'minimum', retryAfterMaxDelayMs: options.maxDelayMs })` call. The old file's 80 lines of header-parsing helpers (`getHeaderValue`, `getResponseHeaderValue`, `hasHeaders`, `hasResponseHeaders`) are deleted because `retryPolicy.ts` now owns that logic.
- New tests at `retry.test.ts:916-1023` cover three previously-untested behaviours: (1) jitter applied inside `persistentCapMs` produces 37,500ms for the construction in the test, (2) `Retry-After` is read from direct headers and honoured, (3) `Retry-After: 600` in interactive mode is capped at the new 300,000ms (5min) ceiling instead of the caller's `maxDelayMs: 1000`.

## Risks / notes

- The interactive-mode `INTERACTIVE_RETRY_AFTER_CAP_MS = 5min` cap is a new behavioural change that's bundled into a "refactor" PR. Previously, a 429 with `Retry-After: 600` in non-persistent mode would block the call for the full 600s; now it caps at 300s and presumably retries early. That's almost certainly more user-friendly, but it's not a pure refactor — should be called out in the PR description.
- The case-insensitive header lookup in the new `retryPolicy.ts` is a strict superset of the old behaviour (the deleted `getRetryAfterDelayMs` was case-sensitive against `'retry-after'` only). A previously-broken edge case (`Retry-After: 3` capitalisation from `response.headers`) now works, as the new test at `retry.test.ts:983-1003` confirms.
- Centralising the math in one helper is unambiguously good. The seven-parameter signature is on the boundary of what's pleasant to call, but it's internal. A small struct/options builder might age better than positional kwargs if more axes get added.
- `getRetryDelayMs` accepts `attempt: 0` (treated as `attempt: 1` via `Math.max(1, attempt)`). The `rateLimit` callers pass `attempt: 1+` already; the in-stream-retry callers in `retry.ts` pass `attempt: 1` as a fixed value. Worth a comment that the helper is 1-indexed to avoid future confusion.

## Verdict

**merge-after-nits** — the refactor is a clear net win in maintainability, the test coverage is genuine (not just port-of-old-tests), and the behavioural shifts are improvements. Update the PR description to call out the new 5-minute interactive cap and the case-insensitive header fix as user-visible changes, and the 1-indexed `attempt` semantics deserve a doc-comment line.
