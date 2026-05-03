# QwenLM/qwen-code PR #3800 — feat(core): support reasoning effort 'max' tier (DeepSeek extension)

- Author: wenshao
- Head SHA: `9c56ec15a15d1aad6d62691804461472be2dedfb`
- Diff: +516 / -57 across 10 files
- Files: docs (model-providers.md, settings.md), `anthropicContentGenerator.{ts,test.ts}`, `contentGenerator.ts`, `geminiContentGenerator.{ts,test.ts}`, `openaiContentGenerator/pipeline.ts`, `openaiContentGenerator/provider/deepseek.{ts,test.ts}`

## Observations

1. **`anthropicContentGenerator.ts:42-62` introduces `isDeepSeekAnthropicHostname`** as a strict hostname-only detector, separate from the broader `isDeepSeekAnthropicProvider` (which falls back to model-name matching for self-hosted sglang/vllm). The split is *the* design decision worth preserving — and the JSDoc at `:80-87` correctly explains why a model-name false-positive is dangerous for `'max'` clamping (it could send `'max'` to real `api.anthropic.com` and trigger HTTP 400).
2. **The clamp test at `anthropicContentGenerator.test.ts:987-1025`** ("still clamps effort: 'max' when model name says 'deepseek' but hostname is api.anthropic.com") is the exact regression case the hostname split exists to prevent. This test would fail if anyone "simplified" `isDeepSeekAnthropicHostname` back to the broader check. Excellent guard.
3. **Budget-token consistency** (`anthropicContentGenerator.test.ts:1027-1067`): clamp from `'max'` to `'high'` must also drop `thinking.budget_tokens` from 128_000 to 64_000, otherwise the wire request is internally inconsistent and Anthropic rejects it. The test asserts both halves of the clamp simultaneously. Good.
4. **`docs/users/configuration/model-providers.md:484-547`** is unusually thorough — the per-provider table calls out exactly which knob each protocol expects (flat `reasoning_effort` for DeepSeek, nested `reasoning: { effort, ... }` for OpenAI-compatible others, `output_config: { effort }` for Anthropic, `thinkingConfig.thinkingLevel` for Gemini). This is the kind of cross-protocol documentation that prevents real on-call pages.
5. **The `samplingParams` interaction warning** (`model-providers.md:519-527`) is a sharp footgun documented sharply: setting `samplingParams` on an OpenAI-compatible provider causes the pipeline to ship those keys verbatim and *silently drop* the top-level `reasoning` field. Anyone configuring a custom DeepSeek deployment with both `samplingParams` and `reasoning` would lose the latter without warning. **Strong recommendation**: at minimum, add a `console.warn`/`debugLogger.warn` in the OpenAI pipeline when both are present and `reasoning` is being silently dropped. Documentation alone won't catch this.
6. **`settings.md` table entry for `model.generationConfig`** got a giant inline blob describing the new `reasoning` field. The table cell is now ~400 chars wide, which makes the rendered docs page hard to scan. Consider promoting `reasoning` to its own dedicated row in the settings table with a short description and a link to the model-providers.md section.

## Verdict: `merge-after-nits`

Substantively correct and well-tested. Two non-blocking nits worth addressing before merge: (a) emit a runtime warning when `samplingParams` causes silent `reasoning` drop on the OpenAI pipeline, and (b) split the new `reasoning` description out of the giant `generationConfig` table cell in `settings.md` for readability.
