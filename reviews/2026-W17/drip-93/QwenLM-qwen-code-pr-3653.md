---
pr: 3653
repo: QwenLM/qwen-code
sha: bb2f3dc46885f810e8abc1678350cc1fe2abad9b
verdict: merge-as-is
date: 2026-04-27
---

# QwenLM/qwen-code #3653 — refactor(config): dedupe QWEN_CODE_API_TIMEOUT_MS env override logic

- **Author**: B-A-M-N
- **Head SHA**: bb2f3dc46885f810e8abc1678350cc1fe2abad9b
- **Link**: https://github.com/QwenLM/qwen-code/pull/3653
- **Size**: +197/-32 across `packages/core/src/models/modelConfigResolver.ts` and `modelConfigResolver.test.ts`.

## Scope

Pure-refactor follow-up to #3629. The `QWEN_CODE_API_TIMEOUT_MS` env-override block was duplicated across `resolveModelConfig()` and `resolveQwenOAuthConfig()`. Per the PR body, #3629 fixed the original OAuth-path bug where the env override wasn't applied; this PR extracts the now-shared logic into one helper without changing behaviour, and adds 8 tests (4 `[Regression]` + 4 `[Additional]`) to lock the precedence in place.

## Specific findings

- `modelConfigResolver.ts:108-130` — new `applyTimeoutEnvOverride(env, generationConfig, sources, modelProvider)` helper. Mutates `generationConfig` and `sources` in-place. Early returns: (1) if `modelProvider.generationConfig.timeout` is already set (modelProvider wins), (2) if `env['QWEN_CODE_API_TIMEOUT_MS']` is `undefined`. Otherwise parses to `Number`, checks `Number.isFinite(parsed) && parsed > 0`, applies `Math.floor(parsed)`. The precedence comment in the JSDoc (`modelProvider > env > settings > default`) matches the runtime behaviour exactly.
- `modelConfigResolver.ts:273-275` — old 14-line block in `resolveModelConfig()` collapses to one call: `applyTimeoutEnvOverride(env, generationConfig, sources, modelProvider)`. Identical surface contract.
- `modelConfigResolver.ts:354-356` — old block in `resolveQwenOAuthConfig()` (variable was named `modelProviderSetTimeoutOAuth` / `envTimeoutOAuth`) also collapses to one call. The duplication that #3629 introduced is now gone. The old per-path variable names existed only because the two blocks lived in different scopes; with extraction, no such namespacing is needed.
- `modelConfigResolver.test.ts:688-722` — `[Regression] OAuth path must apply QWEN_CODE_API_TIMEOUT_MS (was broken before fix #3629)` — directly guards the original bug. Asserts both `result.config.timeout === 45000` and `result.sources['timeout'].kind === 'env'` with `envKey === 'QWEN_CODE_API_TIMEOUT_MS'`.
- `modelConfigResolver.test.ts:723-740` — non-OAuth regression test, sibling of the OAuth one.
- `modelConfigResolver.test.ts:741-779` — `[Regression] modelProvider timeout must win over env` covers both `USE_OPENAI` and `QWEN_OAUTH` paths in the same test, locking down the `modelProvider > env` precedence on both.
- `modelConfigResolver.test.ts:780-800` — `[Regression] env > settings` precedence test on the non-OAuth path.
- `[Additional]` block at `:802-865` — exercises four edge cases against `Number()` parsing: scientific notation (`'1.5e5'` → 150000), hex (`'0x2BF20'` → 180000), empty string (falls through to settings), and a "every supported auth type" loop covering `USE_OPENAI` + `USE_ANTHROPIC`. The hex/scientific-notation assertions are interesting because they're testing `Number()`'s permissive behaviour — fine to lock that in if the PR is a pure refactor with zero behaviour change.
- The comment "// Precedence: modelProvider > env > settings > default (CLI doesn't set timeout)" survives at the call site as a brief one-liner; the longer rationale moved into the helper's JSDoc. Good.

## Risks

The behaviour-preserving claim holds: the inlined logic and the helper produce identical outputs for every input the four-axis test matrix exercises (auth-type × modelProvider-set-or-not × env-set-or-valid × settings-fallback). The shared mutation of `sources['timeout']` is in-place, which matches the original code's pattern. No subtle aliasing or copy-on-write issues since both `generationConfig` and `sources` are object references explicitly threaded through.

## Verdict

**merge-as-is** — exactly the kind of refactor PR that should be merged: behaviour-preserving, removes duplication, and adds a regression test for the original bug it descends from. The 8 tests turn what was previously implicit precedence into explicit, locked-down behaviour. Bonus points for splitting feature/bugfix from refactor.
