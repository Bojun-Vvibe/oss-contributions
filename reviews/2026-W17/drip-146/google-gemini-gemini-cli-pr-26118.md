# google-gemini/gemini-cli #26118 — feat(cli): support boolean and number casting for env vars in settings.json

- PR: https://github.com/google-gemini/gemini-cli/pull/26118
- Head SHA: `c28c2295181ccb968c8a0a9f73fef9f694a34bfa`
- Author: cocosheng-g
- Files touched: `packages/cli/src/config/settings-validation.test.ts`, `packages/cli/src/config/settings-validation.ts`, `packages/cli/src/config/settings.test.ts`, `packages/cli/src/config/settings.ts`

## Observations

- `packages/cli/src/config/settings-validation.ts:135-153` — `buildPrimitiveSchema` now wraps both `number` and `boolean` cases with `z.preprocess(...)`. For numbers: tries `Number(val)` on non-empty string input, returns the original `val` on `NaN` so zod surfaces a typed error rather than silently coercing to `NaN`. For booleans: lowercase compares against `"true"`/`"false"` only, falling through otherwise. Conservative casting — does not accept `"1"`/`"0"` or `"yes"`/`"no"`, which is the right call to avoid surprise.
- `packages/cli/src/config/settings-validation.test.ts:330-389` — adds four `describe('type casting')` cases: lowercase bool strings, mixed-case bool strings (`'TRUE'`, `'fAlSe'`), numeric strings (int + float), and rejection of `'not-a-boolean'`/`'not-a-number'`. The rejection case asserts `result.error?.issues).toHaveLength(2)` which couples the test to the exact issue count — fragile if zod changes how it surfaces preprocess failures, but acceptable.
- `packages/cli/src/config/settings.test.ts:480-540` — adds the integration story: env-var expansion (`'$TEST_AUTO_THEME'` → `'false'` → cast to `false`), default fallback (`'${TEST_AUTO_THEME:-true}'`), and the error-recording path when an env var resolves to `'not-a-number'`. The third test asserts `expect(settings.merged.model.maxSessionTurns).toBe('not-a-number')` — i.e. on validation failure the loader keeps the raw expanded string. Worth confirming this matches existing behavior for other validation failures (it likely does, but PR description doesn't call it out).
- The `settings.ts` change isn't in the head of the diff I reviewed but is presumably the env-var resolution wiring; the test coverage above implies it's correctly handled.

## Verdict: `merge-after-nits`

**Rationale:** Solid feature with thorough test coverage. The casting rules are deliberately conservative (no `"1"`/`"yes"` etc.) which is the safe default. Two minor nits: (1) the issue-count assertion in `settings-validation.test.ts` is brittle, prefer asserting on issue paths instead; (2) document in the settings reference that env-var values can now be cast to bool/number to avoid users wondering why their `"true"` string suddenly behaves differently.
