# cline/cline#10384 — fix: cap retry-after delay to prevent silent multi-hour hangs

- **Head**: `65b1c5e2993f0cc7b8e3d1cc9b06300d57120b3e`
- **Size**: +68/-7 across 2 files
- **Verdict**: `merge-as-is`

## Context

Fixes #10139. When a server returned a large `retry-after` header (e.g. 3 hours from a Gemini quota-exhaustion 429), the `withRetry` decorator at `src/core/api/retry.ts` silently `setTimeout`'d for the full duration with no UI feedback — turning a quota error into a multi-hour hang where the user has no signal anything went wrong. This PR adds a `maxRetryAfter` option (default 60s) that throws the error immediately when the computed delay exceeds the cap.

## Design analysis

The behavior change at `src/core/api/retry.ts:56-71` lands inside the existing 429-handling block, gating only the `retry-after`-header-driven path:

```ts
if (retryAfter) {
    // Handle both delta-seconds and Unix timestamp formats
    const retryValue = Number.parseInt(retryAfter, 10)
    if (retryValue > Date.now() / 1000) {
        // Unix timestamp
        delay = retryValue * 1000 - Date.now()
    } else {
        // Delta seconds
        delay = retryValue * 1000
    }

    if (delay > maxRetryAfter) {
        throw error
    }
} else {
    // Use exponential backoff if no header
    delay = Math.min(maxDelay, baseDelay * 2 ** attempt)
}
```

Three things to confirm about the design:

1. **The cap fires only on `retry-after`-driven delays**, not on the exponential-backoff path. That's the right discrimination — the exponential backoff path is already capped by `maxDelay` (default 10s), so it can't produce multi-hour delays. The pathology was specifically server-driven `retry-after` values.
2. **Default 60s is conservative.** Most CI/automated jobs and interactive agent sessions can tolerate a 60s wait but not 3 hours. Tuned-up callers (e.g. a long-running batch job that genuinely *does* want to wait through a 30-minute window) can pass `maxRetryAfter: 30 * 60 * 1000`. The escape hatch via the new `RetryOptions` field is appropriate.
3. **`throw error` re-raises the original error** at `:69`, preserving the upstream `status: 429` and any `headers["retry-after"]` for caller-side surfacing. That's the right thing — UI layer can present "rate-limited, try again in 3h" with the actual server-provided value, vs the old behavior of silently waiting.

## Test coverage

Two new tests at `src/core/api/retry.test.ts:181-235`:

- `should throw immediately when retry-after exceeds maxRetryAfter` — sets `retry-after: "10800"` (3 hours) and `maxRetryAfter: 60_000`, asserts the original error is thrown and `callCount === 1` (no retry happened). Solid pin.
- `should still retry when retry-after is within maxRetryAfter` — sets `retry-after: "0.01"` (10ms), asserts `callCount === 2`. Confirms the cap doesn't break the normal path.

The existing `should respect retry-after header with Unix timestamp` test at `:113-145` was correctly updated to pass `maxRetryAfter: Infinity` because the synthetic `2010-01-01` timestamp it constructs produces a delay that would now exceed the default 60s cap. But the rewrite at `:114-117` also swapped the date construction to a `sinon.stub(Date, "now")`-based fake — which is a strict improvement (deterministic, no real-clock drift in CI), and the assertion on `delay?.should.equal(1000)` at `:144` is consistent with the new fixture math.

## Concerns

1. **PR template's last two checklist items are unchecked**:
   - "Existing behavior preserved: exponential backoff, maxDelay, maxRetries, non-429 errors all unchanged"
   - "New guard only activates when retry-after header produces a delay > maxRetryAfter"

   Both are true by reading the diff (the cap is gated inside `if (retryAfter) { … }`), but the unchecked boxes suggest the author didn't manually re-verify, which is a small process hygiene nit.
2. **`Number.parseInt` swap from `parseInt`** at `:61` is a free biome-lint cleanup but is technically unrelated to the cap fix. Not enough to demerit, but worth a one-line callout in the PR body to keep the diff legible.
3. **No backoff override.** If a server consistently returns a `retry-after` slightly above the cap (say `retry-after: 65` against the 60s default), every retry attempt throws and the user sees a hard fail — but if `maxRetryAfter` were treated as "wait this long, then retry anyway" the user would get one more try. The "throw immediately" semantics are correct for the multi-hour case but a *little* harsh for the borderline case. Not a blocker (the user can always re-issue), and the alternative semantics make `maxRetryAfter` overlap with `maxDelay`. The current contract is internally consistent.

## Verdict reasoning

Fix is small, targeted at a real "no UI feedback for hours" UX bug, gated correctly so it can't change exponential-backoff behavior, with two clean tests pinning the cap-fires and cap-doesn't-fire branches plus an existing test correctly updated to keep its synthetic-timestamp logic working under the new default. Unchecked PR-template boxes and the unrelated `parseInt`→`Number.parseInt` are paper-cuts, not real concerns.

## What I learned

"Server-driven backoff with no upper bound" is a recurring footgun across rate-limit handling code. The two failure shapes are (a) silent multi-hour hang with zero signal — this PR's case — and (b) tight retry loops when the server returns `retry-after: 0` or omits the header. Both want the same architectural answer: a configurable cap with an explicit-throw fallback when the cap is exceeded, so callers can choose between "wait quietly" and "surface to user." This PR does the right thing for (a); cline's existing exponential-backoff path already handles (b) via `Math.min(maxDelay, …)`.
