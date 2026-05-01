# google-gemini/gemini-cli #26306 — fix(core): prevent infinite retry loop on persistent backend errors

- **Repo:** google-gemini/gemini-cli
- **PR:** https://github.com/google-gemini/gemini-cli/pull/26306
- **HEAD SHA:** `9e88f73`
- **Author:** masqquerade
- **Verdict:** `merge-after-nits`

## What the diff does

Adds an upper bound on how many times `retryWithBackoff` will *reset* its attempt counter after invoking the `onPersistent429` fallback (or after a `ValidationRequiredError` reverify). Before this PR, every successful fallback-model swap reset `attempt = 0` and `currentDelay = initialDelayMs` and continued — so a backend returning persistent 5xx for *both* the primary and the fallback model produced an infinite loop of "retry primary N times → fallback → retry fallback N times → fallback returns same model → retry N times again → …", with the user seeing nothing but ever-spinning UI until they killed the process.

Implementation at `packages/core/src/utils/retry.ts:21,28,46,235,257-258`:

- New constant `DEFAULT_MAX_FALLBACK_COUNT = 3`.
- New required `RetryOptions.maxFallbackCount: number` (default 3).
- New `let resetCount = 0` counter at `:257-258` next to `let attempt = 0`.
- Three reset sites guarded with `if (resetCount >= maxFallbackCount) { debugLogger.warn(...); throw classifiedError; }` and `resetCount++` immediately before `attempt = 0; currentDelay = initialDelayMs; continue;`:
  - The persistent-quota fallback at `:308-326`.
  - The validation-required reset at `:340-356`.
  - The max-attempts-reached fallback at `:376-394`.

The conditional throw at the third site (`:380-386`) preserves the prior shape "throw `RetryableQuotaError` if classified as such, else throw the original `error`" — load-bearing for callers that pattern-match on the error class.

Test at `retry.test.ts:742-764` constructs a mock that always throws HTTP 500 with a fallback callback that always returns `'gemini-2.5-flash'`, runs `retryWithBackoff(... maxAttempts: 2 ...)`, and asserts `fallbackCallback.toHaveBeenCalledTimes(DEFAULT_MAX_FALLBACK_COUNT)` plus the promise rejects with the original error — i.e. the fallback fired exactly 3 times then bailed.

## Why it's right

- **The diagnosis is correct and the bound is the right primitive.** The cycle has two driving inputs (attempt-budget exhaustion *and* `onPersistent429` deciding to swap) and the prior shape only bounded one of them. A second, orthogonal counter (`resetCount`) closes the loop.
- **Three guard sites, one variable.** All three reset sites in `retry.ts` share the same `resetCount`, so a caller can't escape the bound by hopping between quota errors, validation errors, and max-attempts paths. This is the right shape — without the shared counter, `validation→quota→max-attempts` could interleave to a 9-cycle bound.
- **Increment-after-success-only**: `resetCount++` runs *after* the fallback callback returned a non-null model (or the user verified). A failed fallback (returns null/throws) does not consume budget. Correct policy.
- **The conditional throw** (`classifiedError instanceof RetryableQuotaError ? classifiedError : error`) at `:382-385` preserves the prior thrown-type contract — without it, callers handling the quota case specifically would silently start receiving raw transport errors.
- **`debugLogger.warn`** message names the cause ("Exhausted N fallback attempts. Aborting to prevent infinite loop.") — exactly the diagnostic an operator chasing a "why did this just fail after 30 minutes" complaint needs.
- **Test uses `vi.runAllTimersAsync()`** in parallel with the rejection assertion — the right pattern for fake-timer code that needs the backoff sleep to complete without actual wall-clock waits.

## Nits

1. **Three near-identical guard blocks** at `:308-326`, `:340-356`, `:376-394` — extract a `tryReset(reason: 'persistent-quota' | 'validation' | 'max-attempts'): boolean` helper that returns false if budget exhausted (and logs+throws) and true after incrementing. Three sites copy-paste the pattern today; one will drift first.
2. **`maxFallbackCount = 3` default** is unjustified in the diff. Add a one-line code comment explaining the choice (e.g. "primary→flash→pro→flash exhausts the meaningful fallback chain in our routing graph; beyond that we're cycling").
3. **Test coverage gap**: the validation-required path (`:340-356`) is not tested with the new bound. Parametrize the test or add a sibling `should not loop infinitely on persistent ValidationRequiredError`.
4. **`resetCount` is shared across all three reset reasons** but the warning message hard-codes "fallback attempts" — when the third site throws because of validation-only loops the message will mislead operators. Use the per-site message variant ("validation reset budget exhausted" vs "fallback budget exhausted") or include the reason in the log.
5. **`maxFallbackCount` is required** (no `?:`) on the public `RetryOptions` interface — confirm this isn't a minor breaking change for external callers building a `RetryOptions` literal. If it is, make it optional and `?? DEFAULT_MAX_FALLBACK_COUNT` at the destructure.
6. **`onPersistent429` returning the same model the caller is already on** is the actual loop driver in many real backends; consider an early-out at the call site if `fallbackModel === currentModel`. Belt-and-suspenders alongside the bound.

`merge-after-nits` — right primitive (orthogonal `resetCount` shared across all three reset sites), preserved error-class contract, dispositive single-cycle test, but wants the helper extraction, default justification, and validation-path test before this lands as the new floor.
