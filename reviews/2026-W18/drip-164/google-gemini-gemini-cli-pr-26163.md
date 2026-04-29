# google-gemini/gemini-cli #26163 — fix(core): distinguish fallback chains and fix maxAttempts for auto vs explicit model selection

- **PR:** https://github.com/google-gemini/gemini-cli/pull/26163
- **Head SHA:** `f5ede9d35534a41a6b50d297b56816e9e1d96a79`
- **Size:** +251 / −22 (12 files)

## Summary
Two bugs fixed in one PR, both in the model-availability/fallback subsystem:
1. **Explicit model selection was being silently routed through auto-routing chains.** Users picking `--model gemini-3-pro-preview` inherited the auto-mode 3-attempt limit and silent Flash downgrade; this PR introduces an `isAutoSelection` flag through `policyCatalog.ts` so explicit selections get the default 10-attempt + prompt-before-fallback policy.
2. **`geminiChat.ts` resolved `maxAttempts` once at turn-start.** When Pro fell back to Flash mid-turn, Flash inherited Pro's 3-attempt limit instead of its own 10. This is fixed by re-resolving `currentMaxAttempts` per `retryWithBackoff` iteration via the existing `getAvailabilityContext` callback.

Plus an `AUTO_ROUTING_OVERRIDES` constant to dedupe the auto-mode policy fragment, and a `maxAttempts` field on `ModelPolicy` so the per-model override can flow through.

## Specific observations

1. **`policyCatalog.ts:59-66`** introduces `AUTO_ROUTING_OVERRIDES = { maxAttempts: 3, actions: { ...DEFAULT_ACTIONS, transient: 'silent' }, stateTransitions: { ...DEFAULT_STATE, transient: 'sticky_retry' } }` — clean dedup of what was previously inline-spread at two call sites. The explicit-vs-auto branch at `:97-119` then conditionally applies these overrides via `...(isAuto ? AUTO_ROUTING_OVERRIDES : {})`. The asymmetry where the **preview** branch (`:97-110`) inlines the override object instead of using the constant is inconsistent — the preview branch should also reference `AUTO_ROUTING_OVERRIDES` for consistency, otherwise a future tweak (e.g. `maxAttempts: 4`) drifts in one site silently. **Drift risk.**

2. **`modelAvailabilityService.ts:55, 73` adds `attempts: number = 1` to `markRetryOncePerTurn` and stores it in the `sticky_retry` state.** The default of `1` preserves prior behavior, and `selectFirstAvailable` at `:108-111` reads back `state.attempts` instead of the previous hardcoded `1`. The `attempts` field flows correctly from `policy.maxAttempts ?? 1` through `applyModelSelection`. Test at `modelAvailabilityService.test.ts:37-41` (`tracks retry with custom attempts`) pins the new behavior with `markRetryOncePerTurn(model, 3)` → `expect(selection.attempts).toBe(3)`. Good.

3. **`errorClassification.ts:15-40` widens each `instanceof` check with a structural fallback** — `(error && typeof error === 'object' && 'name' in error && error.name === 'TerminalQuotaError')`. This is a workaround for the dual-instance problem (multiple bundled copies of the error class across npm-link / vitest module-graph splits causing `instanceof` to return false for what is structurally the same error). The pattern is correct but **brittle**: any future rename of `TerminalQuotaError`/`RetryableQuotaError`/`ModelNotFoundError` requires updating the string literal in three places. A `static readonly NAME = 'TerminalQuotaError'` on each error class + comparing `error.name === SomeError.NAME` would chain rename safety. **Minor.**

4. **`fallbackIntegration.test.ts:80-113`** is the load-bearing regression test for the maxAttempts fix — it explicitly asserts `result1.maxAttempts === 3` (Pro), then triggers `consumeStickyAttempt`, asserts `result2.model === FLASH_MODEL` (fallback), then `resetTurn()` and asserts `result3.maxAttempts === 3` (Pro restored). This catches the per-turn-resolution bug. However, **it does not catch the per-iteration resolution bug** that `geminiChat.ts` is supposed to fix — the test exercises `applyModelSelection` (which is the policyHelpers entry point) but not the `retryWithBackoff` iteration loop. A test that mocks a Pro→Flash fallback mid-loop and asserts Flash gets its full 10 attempts (not Pro's 3) would lock the actual claimed fix. The PR description points to `geminiChat.test.ts` for that — worth confirming a test there exists.

5. **`policyCatalog.test.ts:56-62`** has a meaningful semantic update: the old test asserted `previewPolicy.stateTransitions.transient === 'terminal'` for *any* preview chain, the new test asserts `=== 'sticky_retry'` only when `isAutoSelection: true`. The implication: an explicit preview-model selection now treats transient errors as terminal (no auto-retry) — that's a **behavioral change** for users on `--model gemini-3-pro-preview`. They previously got silent retries; now they get a hard fail with a prompt. This is the intended fix per the PR description ("explicit selections... prompt before fallback"), but it's worth a release-note since affected users will see a UX shift.

## Verdict: `merge-after-nits`

The diagnosis (auto-routing-policy was leaking into explicit selection + maxAttempts was statically resolved) is sharp and the fix is structurally sound. The two bugs are real and the test surface for one of them is good. The drift risk on the preview-branch override duplication and the per-iteration test coverage gap are the things that should land before merge.

## Recommended actions
- **(Required)** Use `AUTO_ROUTING_OVERRIDES` in the preview branch at `policyCatalog.ts:97-110` instead of inlining the same fragment — eliminates drift risk for the next policy change.
- **(Required)** Confirm `geminiChat.test.ts` actually has a mocked Pro→Flash mid-turn fallback test that asserts Flash gets its full 10 attempts. The fallbackIntegration test only covers the policy-resolver entry point, not the retry-loop iteration that the PR description claims to fix.
- **(Nit)** Replace the duplicated string literals in `errorClassification.ts` with `static readonly NAME` constants on each error class for rename-safety.
- **(Required)** Release-note the explicit-preview behavioral shift: users on `--model gemini-3-pro-preview` will now see prompt-on-failure instead of silent Flash downgrade.
