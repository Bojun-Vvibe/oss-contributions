# QwenLM/qwen-code PR #3800 — feat(core): support reasoning effort 'max' tier (DeepSeek extension)

- **Repo:** QwenLM/qwen-code
- **PR:** #3800
- **Head SHA:** `b5b05ae2194ba195f88e08ceebc9e33536f69cf0`
- **Author:** wenshao
- **Title:** feat(core): support reasoning effort 'max' tier (DeepSeek extension)
- **Diff size:** +366 / -23 across multiple files (docs + core anthropicContentGenerator + tests)
- **Drip:** drip-293

## Files changed

- `docs/users/configuration/model-providers.md` (+~50) — adds a "Reasoning / thinking configuration" section with a per-provider behavior table covering OpenAI/DeepSeek, OpenAI-other, Anthropic real, Anthropic via DeepSeek, and Gemini.
- `docs/users/configuration/settings.md` (+~5/-~5) — adds `reasoning` to the `model.generationConfig` description with cross-link to the new section. Pure docs reflow makes the diff look bigger than it is.
- `packages/core/src/core/anthropicContentGenerator/anthropicContentGenerator.test.ts` (+~76) — two new tests: (1) `passes effort: 'max' through to output_config and bumps thinking budget` asserting `output_config: { effort: 'max' }` and `thinking: { type: 'enabled', budget_tokens: 128_000 }` for a `deepseek-v4-pro` model; (2) `clamps effort: 'max' to 'high' on a non-DeepSeek anthropic provider` asserting clamp behavior against `https://api.anthropic.com`.
- `packages/core/src/core/anthropicContentGenerator/anthropicContentGenerator.ts` (+~25/-~5) — extends the `output_config` type to include `'max'`; adds `'max' → 128_000` to the `budgetTokens` ternary chain; adds `isDeepSeekAnthropicProvider(...)` gate in `buildOutputConfig` that clamps `'max' → 'high'` with a `debugLogger.warn(...)` for non-DeepSeek targets.

## Specific observations

- `anthropicContentGenerator.ts:393-405` — the budget ternary chain is now four-deep (`low → 16k`, `max → 128k`, `high → 64k`, default → 32k). Once you cross 3 ternaries, switch to a `Record<Effort, number>` lookup or `match`/switch — much easier to extend and review. The current ordering also reads oddly (max sandwiched between low and high); a table would normalize it.
- `anthropicContentGenerator.ts:432-446` — clamp-and-warn for non-DeepSeek targets is the right call to avoid 400s when a user moves a config between providers. The `debugLogger.warn(...)` message is clear. One nit: the `isDeepSeekAnthropicProvider(...)` call isn't visible in this diff hunk — verify it does substring matching on `baseUrl` (e.g. `api.deepseek.com/anthropic`) and isn't doing strict equality, which would miss user-provided variants.
- `anthropicContentGenerator.test.ts:487-528` — DeepSeek-target test is precise: pins `output_config: { effort: 'max' }` AND `thinking.budget_tokens: 128_000`. Good.
- `anthropicContentGenerator.test.ts:530-572` — clamp test pins `output_config: { effort: 'high' }` for a `https://api.anthropic.com` target. Missing assertion: the `debugLogger.warn` was actually called. Spy on the logger and assert the warning fires; otherwise the user-visible signal is unverified.
- Missing test: behavior on `baseUrl: 'https://api.deepseek.com/anthropic'` (the DeepSeek-via-Anthropic path) — the docs table claims `'max'` passes through unchanged on that endpoint, but only `deepseek-v4-pro` (which uses the OpenAI client) is exercised. Add a test that hits the Anthropic client *with* a DeepSeek baseUrl.
- Docs are unusually thorough; the per-provider table at `model-providers.md:495-507` is exactly what users need. Verify the Gemini row's claim that `'max'` maps to `HIGH` (since Gemini has no MAX tier) is reflected in the actual Gemini converter — this PR doesn't touch that file, so the doc is making a forward-looking promise.
- Pre-existing settings.md table reflow is huge diff noise (every row's pipe alignment changed). Either land that as a separate "docs: reflow settings table" commit or accept the noise. Reviewers will struggle to spot real changes in that file.

## Verdict: `merge-after-nits`

Solid feature add with correct clamping semantics, sensible budget bump, and exceptional documentation. Nits: (1) refactor the four-deep budget ternary into a lookup table, (2) assert the clamp `debugLogger.warn` actually fires, (3) add a test for the DeepSeek-via-Anthropic baseUrl path, (4) verify the Gemini converter actually honors `'max' → HIGH` as the docs claim, (5) consider splitting the settings.md table reflow into its own commit. The shape is right.
