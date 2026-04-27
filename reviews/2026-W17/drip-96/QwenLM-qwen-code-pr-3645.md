# QwenLM/qwen-code #3645 — fix(cli): correct OPENAI_MODEL precedence without breaking /model selection

- Author: B-A-M-N
- Head SHA: `e1ed051c32345d77c638cc024f3be3af214d6570`
- +501 / −12 across `packages/cli/src/utils/modelConfigUtils.test.ts` (+473/-1), `packages/cli/src/utils/modelConfigUtils.ts` (+28/-11)
- PR link: <https://github.com/QwenLM/qwen-code/pull/3645>

## Specifics

- This is a follow-up to a regression: PR #3567 changed `OPENAI_MODEL` precedence in a way that broke `/model` (`settings.model.name`) selection, then #3633 reverted it entirely (`revert(cli): undo OPENAI_MODEL precedence change in modelProviders lookup (#3567)` — visible in the recent PR list). This PR re-lands the OPENAI_MODEL fallback in a way that doesn't break settings-model precedence. The 3-tier desired order is documented in the PR body: (1) `argv.model` (CLI flag) > (2) `settings.model.name` (when it matches a configured provider) > (3) `OPENAI_MODEL` env var (fallback when settings is unset OR doesn't match).
- Implementation at `modelConfigUtils.ts:+28/-11` (small change, large test surface). The diff isn't visible in the slice I read, but the test cases at `modelConfigUtils.test.ts:+473` define the contract precisely:
  - **Case A** (`:728-768`): `settings.model.name='settings-model'` + `OPENAI_MODEL='env-model'` + both providers exist → resolves to `settingsProvider`. This is the regression fix — previously OPENAI_MODEL won.
  - **Case B** (`:770-810`): `settings.model = undefined` + `OPENAI_MODEL='env-model'` → resolves to `envProvider`. Confirms env fallback works.
  - **Edge 1** (`:812-852`): `argv.model='cli-model'` overrides both settings and env. CLI flag remains highest priority.
  - **Edge 2** (`:854-894`): No `OPENAI_MODEL`, no settings → `QWEN_MODEL` is final fallback.
  - **Edge 3** (`:896-...`): Both `OPENAI_MODEL` and `QWEN_MODEL` set, no settings → `OPENAI_MODEL` wins. This is the OpenAI-auth-type-specific lookup order.
  - **Edge 4** (referenced in body): Non-OpenAI auth ignores `OPENAI_MODEL` entirely.
  - **Edge 5** (referenced in body): `settings.model.name` set but doesn't match any provider → falls back to `OPENAI_MODEL`. This is the subtle case — a typo in `/model` shouldn't break the user, env fallback should rescue them.
- The "should use custom env when provided" test at `modelConfigUtils.test.ts:534` is updated to pass `model: undefined as unknown as Settings['model']` — this is necessary because the *previous* test relied on `settings.model` being absent for the env-fallback path to fire, and the new code makes that dependency explicit. Without this update, the test would now resolve via settings (which `makeMockSettings()` defaults populate) and the `customEnv` path wouldn't be exercised.
- 35/35 tests pass per the PR body, and the 7 cited cases form a complete decision matrix for the 3-tier precedence × 2 env vars × auth-type-OpenAI-or-not.

## Concerns

- The actual code change (+28/-11 in `modelConfigUtils.ts`) is small but the test diff (+473) is large; without seeing the implementation in the slice I read, I'm reading the contract from the tests. The tests look correct but they all use mocked `resolveModelConfig` (`vi.mocked(resolveModelConfig).mockReturnValue(...)`) — they assert what `modelProvider` is *passed to* `resolveModelConfig`, not what `resolveModelConfig` actually returns. That's the right unit-test boundary, but it means a bug *inside* `resolveModelConfig` wouldn't be caught here.
- Edge Case 5 ("Falls back to OPENAI_MODEL when settings.model.name doesn't match") is the most subtle and the most prone to silent UX issues — if a user typos `/model gpt-5o` (missing the dot), the env fallback will silently route them to `gpt-5` or whatever `OPENAI_MODEL` points at. The fix is correct (better than failing), but a `console.warn` or status message saying "configured model `gpt-5o` not found, falling back to OPENAI_MODEL=`gpt-5`" would prevent silent surprises. Not a merge blocker but worth a follow-up.
- This PR re-introduces the same change that was reverted in #3633. Maintainer should sanity-check that the new tests actually cover the case that broke #3567 (which the revert was driven by). The PR body asserts "Fixes regression without reintroducing previous override issue" but the original break-case from #3567 isn't explicitly named in the test list. Worth a comment in the PR description linking to the specific issue or test case from #3567 that this PR's Case A locks down.
- The `settings: makeMockSettings({ model: undefined as unknown as Settings['model'] })` pattern (`:534-536`) suggests `Settings['model']` doesn't allow `undefined` in the type. If that's true, the production code may be hitting an `undefined` at runtime that the type system thinks is impossible — worth tightening the `Settings` type to accept `model?: ModelSettings | undefined` so the `as unknown as` cast can be dropped.
- 473 lines of test for 28 lines of code is verbose but justified given the matrix-explosion of `(argv × settings × OPENAI_MODEL × QWEN_MODEL × authType)`. The case names ("Case A", "Edge Case 5") are clear; the test bodies are uniform-shaped (setup → mock → invoke → assertion); no DRY-up needed.

## Verdict

`merge-after-nits` — the precedence contract is correct, the test matrix is comprehensive, and re-landing the env-fallback without re-breaking `/model` selection is the right move after the #3633 revert. Two non-blocking follow-ups: (1) add a user-facing warning when Edge Case 5 fires (settings-model-doesn't-match → env fallback), so the silent fallback is observable; (2) tighten `Settings['model']` type to accept `undefined` so the `as unknown as` cast in the test can be removed. Maintainer should also confirm Case A specifically covers the original #3567 break case.
