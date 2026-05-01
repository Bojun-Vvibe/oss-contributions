# google-gemini/gemini-cli#26306 — fix(core): prevent infinite retry loop on persistent backend errors

- **PR**: https://github.com/google-gemini/gemini-cli/pull/26306
- **Head SHA**: `723153af7c5f754c14fc6dcf06e61bf2c5fd0d90`
- **Size**: +56 / -1, 2 files
- **Verdict**: **merge-after-nits**

## Context

Liveness fix for `retryWithBackoff` in `packages/core/src/utils/retry.ts`. The previous implementation reset `attempt = 0` on a successful fallback-model switch, but if the fallback model also returned a persistent retryable error (500, RetryableQuotaError, ValidationRequiredError), the catch block would fall through, trigger another fallback, and reset attempts again — an unbounded `while` loop that hangs the CLI during sustained backend outages.

## What's right

**`retry.ts:21-22 / 45-46` — new `DEFAULT_MAX_FALLBACK_COUNT = 3` constant + `maxFallbackCount` field on `RetryOptions`.**

Exporting the constant lets the test suite reference the same value as the production default (visible at `retry.test.ts:12` `import { DEFAULT_MAX_FALLBACK_COUNT, retryWithBackoff }`), so a future bump of the default doesn't silently invalidate the test assertion. Good discipline.

**`retry.ts:259` — `let resetCount = 0;` declared at the top of the retry loop, alongside `let attempt = 0;`.**

State is loop-scoped, not function-scoped, so a fresh `retryWithBackoff` invocation always starts at zero. Correct.

**Three guard insertions, one per `attempt = 0` reset call site.**

1. **`:310-323` — `is500/RetryableQuotaError` arm.** Before the existing `onPersistent429(authType, classifiedError)` await, check `if (resetCount >= options.maxFallbackCount)` and re-throw `classifiedError` if exhausted. After a successful fallback (`if (fallbackModel)`), increment `resetCount++` *before* `attempt = 0`. This is the load-bearing arm for the reported bug (persistent 500s).
2. **`:339-352` — `ValidationRequiredError` arm.** Same pattern. The user-validation flow can also flip into "user keeps verifying, backend keeps rejecting" — same unbounded reset risk.
3. **`:374-394` — max-attempts-exhausted-but-persistent-quota arm.** This is the trickier one: when `attempt >= maxAttempts` but the error is `RetryableQuotaError`, the prior code would still trigger a fallback and reset. The new guard is also there, with the additional subtlety that the thrown error is `classifiedError instanceof RetryableQuotaError ? classifiedError : error` — preserving the original (non-classified) error if classification stripped useful context. Defensible.

**Test — `retry.test.ts:742-763`.**

```ts
it('should not loop infinitely if the fallback model also returns 500', async () => {
  const fallbackCallback = vi.fn().mockResolvedValue('gemini-2.5-flash');
  const mockFn = vi.fn().mockImplementation(async () => {
    const error: HttpError = new Error('Persistent Backend Error');
    error.status = 500;
    throw error;
  });
  const promise = retryWithBackoff(mockFn, {
    maxAttempts: 2,
    initialDelayMs: 1,
    onPersistent429: fallbackCallback,
    authType: AuthType.LOGIN_WITH_GOOGLE,
  });
  await Promise.all([
    expect(promise).rejects.toThrow('Persistent Backend Error'),
    vi.runAllTimersAsync(),
  ]);
  expect(fallbackCallback).toHaveBeenCalledTimes(DEFAULT_MAX_FALLBACK_COUNT);
});
```

Pins the load-bearing contract: callback is invoked exactly `DEFAULT_MAX_FALLBACK_COUNT` times (3), then the original error is thrown rather than the loop continuing. Uses `Promise.all([rejects.toThrow, runAllTimersAsync])` to drive the fake timers forward — the right pattern for vitest fake-timer-based retry tests.

## Risks / nits

- **`resetCount` is shared across all three arms.** A run that does 1 successful 500-fallback → 1 successful validation-fallback → 1 successful 500-fallback consumes all 3 budget across heterogeneous causes. Probably the right semantics (the budget is "number of state-resets", not "number of state-resets of each kind"), but worth a one-line comment on the `let resetCount = 0;` declaration noting "shared across all three reset arms".
- **String formatting at `:312` and `:380`** uses `"Exhausted " + options.maxFallbackCount + " fallback attempts. Aborting to prevent infinite loop."` — the existing surrounding code uses template literals (e.g. `:344` uses backticks). One-line consistency cleanup.
- **Test asserts only the 500/`onPersistent429` path.** The two other arms (`ValidationRequiredError` and the max-attempts-then-quota path) do not get their own regression test. The structural fix is the same so confidence is reasonable, but a parametrized test or two more arms would prevent a future refactor from regressing only one of the three guards.
- **`maxFallbackCount` is now a required field on `RetryOptions`** (no `?`), but the type-level requirement is satisfied by `DEFAULT_RETRY_OPTIONS` providing the default at `:46`. Confirm no external caller is constructing `RetryOptions` literals without spreading `DEFAULT_RETRY_OPTIONS` — if any do, this is a minor breaking change at the type level. A quick grep for `RetryOptions = {` at call sites would confirm.
- **No upper bound on `maxFallbackCount` if a caller overrides it to a large number.** Not really a regression (the prior behavior was effectively `Infinity`) and `DEFAULT_MAX_FALLBACK_COUNT = 3` is sane, but worth a one-line comment in the doc-comment on `RetryOptions.maxFallbackCount` recommending values <10.
- **`debugLogger.warn` messages at `:312`, `:344`, `:380` are user-facing in debug mode.** "Exhausted N fallback attempts. Aborting to prevent infinite loop." is fine; "Exhausted allowed state resets. Aborting to prevent infinite validation loop." is clear. Consistent voice would be nice (one says "fallback attempts", the other "state resets") — pick one and reuse.

## Verdict

**merge-after-nits.** Real liveness fix, correct three-arm coverage, regression test pins the most important path. Cosmetic cleanups (template literals, message-voice consistency, one-line comment on `resetCount` semantics) and a couple more test arms for the validation/max-attempts paths would tighten it up but don't block the merge.
