# google-gemini/gemini-cli PR #26383 — fix(web-fetch): rate limiting in fallback execution

- Link: https://github.com/google-gemini/gemini-cli/pull/26383
- SHA: `441915acfeb244d8992223462b9d623085f5853f`
- Author: Akash504-ai
- Stats: +34 / 0, 2 files (`packages/core/src/tools/web-fetch.ts`, `packages/core/src/tools/web-fetch.test.ts`)

## Summary

Closes #26382 by enforcing the existing `checkRateLimit` host-based rate limiter inside the fallback web-fetch execution path. Previously the limiter was applied only on the primary path, so once the primary failed and execution dropped into `executeFallbackForUrl`, callers could burst the same host without bound. Adds one Vitest case that drives 11 calls through the fallback path and asserts the 11th returns an error.

## Specific references

- `packages/core/src/tools/web-fetch.ts` L285–L292 (after the diff): the new guard runs `checkRateLimit(urlStr)` *before* the existing `isBlockedHost(url)` check. Order is correct — rate limit should reject before any work, including blocked-host log emission. Note `urlStr` is passed to `checkRateLimit` while `isBlockedHost` receives the GitHub-rewritten `url`. That's deliberate (rate limit is per requested host, not per resolved host) but worth a one-line comment so future readers don't "fix" it.
- L285–L292: the thrown error message `Rate limit exceeded for ${urlStr}` is unstructured. The test asserts only `result.llmContent.toContain('Error')` (test L36), so any throw would satisfy it. Consider asserting the actual message substring (`'Rate limit exceeded'`) so a future regression that swallows this into a generic error is caught.
- `web-fetch.test.ts` L672–L693: the test mocks `isPrivateIp` false, primes the primary path to fail via `mockGenerateContent.mockResolvedValueOnce({ candidates: [] })`, then loops 10 successful fetches and a failing 11th. Two test-quality nits: (1) the loop calls `await tool.build(...).execute(...)` without asserting any of those 10 succeeded — if the rate limiter were already broken in the *opposite* direction (rejecting too early), the test would still pass when the 11th errors. Add an assertion on at least one early call. (2) `mockResolvedValueOnce` with `candidates: []` only forces fallback for the *first* call; the next 10 hit the primary path again. The test name "should respect rate limit in fallback execution" doesn't quite match — really the limiter happens to fire on a fallback call only because of state accumulated by primary calls.
- The fix is correct in spirit but the test as written doesn't fully prove "rate limit enforced in fallback specifically" — it proves "rate limit fires after 10 calls regardless of path". A second test that pins every call to the fallback path would make the regression boundary tight.

## Verdict

verdict: merge-after-nits

## Reasoning

The production change is one small, correct guard at the top of `executeFallbackForUrl` and the security/abuse argument for it is sound. The test exists and would fail if the guard were removed, which is the bar. Nits are about test specificity (force *every* call into fallback, assert on the actual error message, assert early calls succeed) and a one-line comment explaining why `urlStr` vs `url` is used in the two checks. None blocking.
