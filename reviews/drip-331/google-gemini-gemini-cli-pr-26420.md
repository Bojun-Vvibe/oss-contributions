# google-gemini/gemini-cli #26420 — fix(core): ignore GOOGLE_CLOUD_PROJECT for LOGIN_WITH_GOOGLE

- SHA: `17a430415ba56e8d3cbbe0df55a84647da4dd60b`
- State: OPEN, +78/-5 across 3 files
- Closes: #19865, #26105

## Summary

For Free Tier `LOGIN_WITH_GOOGLE` users with `GOOGLE_CLOUD_PROJECT` set in `~/.gemini/.env`, the Code Assist API returns 403 because the env-supplied project isn't reachable from their personal account. PR threads `authType` into `setupUser` and adopts a "conditional double-load" strategy: try without project id first; on `ProjectIdRequiredError` or `IneligibleTierError`, retry with the env-supplied project id. Other auth types (`COMPUTE_ADC`) keep strict env-var semantics with no fallback.

## Notes

- `packages/core/src/code_assist/setup.ts:120-126` — `initialProjectId = authType === AuthType.LOGIN_WITH_GOOGLE ? undefined : envProjectId` is the right gate. The asymmetric handling is documented well by the test cases. One small concern: `authType` parameter is optional (`authType?: AuthType`), so any existing caller that omits it gets the *old* behavior (env var honored). Confirms this is non-breaking, but a TODO/jsdoc note that `authType` should become required after caller migration would prevent future regressions.
- `packages/core/src/code_assist/setup.ts:137-152` — the try/catch + `projectCache.getOrCreate(envProjectId, ...)` fallback path correctly *re-caches* under `envProjectId` so a subsequent call with the same env var hits cache instead of re-attempting the no-project load. Good. **Subtle issue**: the outer `projectCache.getOrCreate(initialProjectId, ...)` caches a Promise that itself awaits the inner fallback. If the inner `_doSetupUser(client, envProjectId, ...)` throws (e.g., transient network), the *outer* cache entry under `initialProjectId=undefined` is now a rejected Promise. Future calls with `LOGIN_WITH_GOOGLE` will re-throw the cached rejection without retrying. Consider deleting the outer cache entry on inner failure, or using `cache.set` only on success.
- `packages/core/src/code_assist/setup.test.ts:163-181` — the "fallback to env var" test mocks `mockLoad` twice (reject then resolve) and asserts `mockLoad` is called twice. Solid coverage of the happy fallback path. Missing case: a third test where the **fallback also fails** — current code re-throws the inner error, which is correct, but no test asserts that no infinite-loop or third call happens.
- `packages/core/src/code_assist/setup.test.ts:194-202` — COMPUTE_ADC negative test (no fallback) is precisely the right guardrail. Good.
- `packages/core/src/code_assist/codeAssist.ts:25` — single-line plumbing of `authType` through `setupUser`. No other callers of `setupUser` exist in the diff, but a quick repo search to confirm no third-party callers (or older code paths) is worth a moment.
- `IneligibleTierError` is included in the fallback predicate but not exercised by any new test. Add a fourth test stubbing `mockLoad.mockRejectedValueOnce(new IneligibleTierError())` to cover that branch — otherwise a future refactor that drops it from the predicate would be invisible.

## Verdict

`merge-after-nits` — diagnosis is correct, the fix is minimal and well-scoped, and tests cover the primary behaviors. Address the cached-rejection edge case (or document why it's acceptable), add the missing `IneligibleTierError` test, and clarify in jsdoc that `authType` should be required for new call sites.
