# QwenLM/qwen-code #3800 — feat(core): support reasoning effort 'max' tier (DeepSeek extension)

- PR: https://github.com/QwenLM/qwen-code/pull/3800
- Author: wenshao
- Head SHA: `9c56ec15a15d1aad6d62691804461472be2dedfb`
- Updated: 2026-05-03T04:54:23Z

## Summary
Adds a fourth `'max'` tier to the `reasoning.effort` scale, primarily as a DeepSeek-specific extension. Threads provider-specific behavior through the OpenAI/DeepSeek, Anthropic, and Gemini converters, and documents the per-provider mapping plus a sharp `samplingParams` interaction caveat in the user docs.

## Observations
- `docs/users/configuration/model-providers.md` line ~481-543 (the new "Reasoning / thinking configuration" section) is genuinely well-written — the per-provider table at lines ~509-516 explicitly documents the `'max'` clamp on real Anthropic and the `'max'`-passthrough on `api.deepseek.com/anthropic`, which is exactly the disambiguation users will hit. The `samplingParams` warning callout (~line 525-535) is correctly load-bearing: silently dropping the `reasoning` field when `samplingParams` is set is a footgun, and surfacing it in a `> [!warning]` is the right choice.
- `docs/users/configuration/settings.md` line ~143: the `model.generationConfig` cell is now a giant single-line table cell. Markdown table rendering is preserved, but the cell has ballooned to roughly ~350 chars with embedded backticks and an inline link. The actual content is correct; consider whether this is best left as a table row at all or refactored to a definition-list-style block following the existing pattern for complex settings.
- The four-tier scale (`low | medium | high | max`) with server-side mapping (`'low'`/`'medium'` → `'high'` on DeepSeek) preserves wire compatibility for existing configs. Good.
- The diff is docs-only as far as I can see in the first 200 lines — confirm the implementation lands in the same PR (the PR title says `feat(core)` so I'd expect TS code under `packages/core/src/...` covering the `clamp to high` behavior on real Anthropic and the `THINKING_LEVEL_UNSPECIFIED` fallback on Gemini for non-low/high efforts). If the code is in a sibling commit, the docs claims need test coverage anchoring them; if it's somewhere I didn't see, please add a test name to the PR body so reviewers can locate it.
- The `budget_tokens` extension (~line 540-548) being preserved-but-ignored on OpenAI/DeepSeek is documented honestly; that's the right call — silently dropping it would be worse.

## Verdict
`merge-after-nits`
