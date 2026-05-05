# sst/opencode PR #25919 — fix(opencode): default temperature for openai-compatible config models

- URL: https://github.com/anomalyco/opencode/pull/25919
- Head SHA: `0809cac77aecf43279e04d6aa0494ea97317b8ed`
- Size: +5 / -1

## Summary

Single-line behavior tweak in `packages/opencode/src/provider/provider.ts:1206` that changes the default value of `capabilities.temperature` from a hard `false` to a conditional based on `apiNpm`: if the upstream package is `@ai-sdk/openai-compatible`, default to `true`; else preserve the previous `false`. Test `packages/opencode/test/provider/provider.test.ts:300` now asserts `temperature === true` for a custom provider declared with the `openai-compatible` npm package.

## Specific findings

- `provider.ts:1203-1212` — fallback chain is `model.temperature ?? existingModel?.capabilities.temperature ?? (apiNpm === "@ai-sdk/openai-compatible" ? true : false)`. Order is correct: explicit per-model config wins, then any pre-existing model record, then the new conditional default. So users who already set `temperature: false` for a quirky openai-compatible endpoint (e.g. an o1-style reasoning proxy) keep working.
- `provider.test.ts:300` — adds one assertion line inside an existing "custom provider with npm package" test. The fixture for that test (not shown in the diff but referenced by the existing `expect(...models["custom-model"]).toBeDefined()` at `:299`) declares the npm package as `@ai-sdk/openai-compatible`, so the assertion exercises the new branch directly.

## Notes

- Strict equality on the literal string `"@ai-sdk/openai-compatible"` is brittle if the AI-SDK ever renames or namespaces (e.g. `@ai-sdk/openai-compatible-v2`), but matches existing string-keyed adapter dispatch elsewhere in this file. Acceptable.
- A small inline comment naming *why* openai-compatible defaults to `true` (i.e. "Almost every OpenAI-API-compatible endpoint accepts `temperature`; legacy default of `false` was muting it") would help future readers, but not a merge blocker.
- No regression coverage for "explicit `model.temperature: false` still wins" — would be a useful one-liner addition, but the precedence is obvious from the `??` chain so this is a nit.

## Verdict

`merge-after-nits`
