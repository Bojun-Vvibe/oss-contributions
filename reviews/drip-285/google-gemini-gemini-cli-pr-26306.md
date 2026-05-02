---
repo: google-gemini/gemini-cli
pr: 26306
head_sha: a6ef68ac61db583b75959de5a0568123a04a3b76
title: "fix(core): prevent infinite retry loop on persistent backend errors"
verdict: merge-after-nits
reviewed_at: 2026-05-03
---

# Review: google-gemini/gemini-cli#26306 — `prevent infinite retry loop on persistent backend errors`

**Head SHA:** `a6ef68ac61db583b75959de5a0568123a04a3b76`
**Stat:** +57 / −1 across `packages/core/src/utils/{retry.ts, retry.test.ts}`.

## Bug

`retryWithBackoff` reset `attempt` to 0 every time `onPersistent429`
swapped in a fallback model. If the fallback *also* returned errors that
qualified as "persistent" (e.g. 500), the swap would re-fire, the
counter would re-zero, and the loop would never terminate. Users hit
this as the CLI hanging indefinitely during backend outages.

## Fix

Two changes in `retry.ts`:

1. New constant + option (around lines 20–47):

```diff
 export const DEFAULT_MAX_ATTEMPTS = 10;
+export const DEFAULT_MAX_FALLBACK_COUNT = 3;

 export interface RetryOptions {
   maxAttempts: number;
+  maxFallbackCount: number;
   ...
 }

 const DEFAULT_RETRY_OPTIONS: RetryOptions = {
   maxAttempts: DEFAULT_MAX_ATTEMPTS,
+  maxFallbackCount: DEFAULT_MAX_FALLBACK_COUNT,
   ...
 };
```

2. The destructuring in `retryWithBackoff` (around line 235–257) pulls
out `maxFallbackCount`, and the fallback handler now tracks how many
times the fallback has been triggered. When that count reaches
`maxFallbackCount`, the next persistent error is *not* swallowed by
another fallback — it propagates out as the rejection.

The new test (`retry.test.ts:742–762`) makes the regression explicit:

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

The `toHaveBeenCalledTimes(DEFAULT_MAX_FALLBACK_COUNT)` assertion is
exactly the right shape — it pins the bound, not just "doesn't hang".

## Assessment

- Liveness fix for a real production hang. Default of 3 fallback swaps
  is conservative and matches the spirit of "retry, but don't retry
  forever".
- Test uses `vi.runAllTimersAsync()` correctly to drain the backoff
  delays without making the test slow.
- API addition (`maxFallbackCount` in `RetryOptions`) is non-breaking
  because of the spread-with-defaults pattern at line 245-ish.

## Nits

1. **`RetryOptions` is now a strict-required field.** The interface
   declares `maxFallbackCount: number;` (not optional). Any external
   caller constructing a `RetryOptions` literal directly (rather than
   relying on `DEFAULT_RETRY_OPTIONS`) will break at compile time.
   Either mark it `?: number` or document this as a deliberate
   breaking change in the package changelog. Same comment applies
   structurally to the existing `maxAttempts` field, so this just
   matches existing style — but it's worth being deliberate about.

2. **Naming.** `onPersistent429` now also handles 500s
   (per the new test). The handler name has been a misnomer for a
   while; this PR cements that. Worth a follow-up rename to
   `onPersistentRetryableError` or similar so future maintainers
   don't get confused. Out of scope here.

3. **Test could go one further.** Asserting that the rejection's
   `.cause` (or wrapped error chain) preserves which model was the
   last-attempted fallback would help users debug "which fallback
   actually died". Nice-to-have.

## Verdict

**`merge-after-nits`** — correct bug fix, right test, narrow blast
radius. Nit #1 (interface optionality) is the only one I'd want a
maintainer to consciously decide on before landing; the others are
follow-ups.
