# sst/opencode PR #24500 — fix(docs): correct OpenCode Go DeepSeek endpoints

- Repo: sst/opencode
- PR: #24500
- Head SHA: `5bb151c1`
- Author: @MrMushrooooom
- Diff: +36/-36 across 18 files (all under `packages/web/src/content/docs/<lang>/go.mdx`)

## What changed

Pure docs fix that flips the two `DeepSeek V4 Pro` and `DeepSeek V4 Flash` rows in every localized `go.mdx` table from the Anthropic-compatible `/v1/messages` endpoint + `@ai-sdk/anthropic` SDK column over to the OpenAI-compatible `/v1/chat/completions` endpoint + `@ai-sdk/openai-compatible` column. Same two-line table edit replicated identically across all 18 language variants (ar, bs, da, de, …).

## Specific observations

- The diff is mechanical — every hunk is the exact same two-line swap at the `DeepSeek V4 Pro` / `DeepSeek V4 Flash` rows of the "API endpoints" table (e.g. `packages/web/src/content/docs/ar/go.mdx:143-144`, `packages/web/src/content/docs/bs/go.mdx:155-156`, `packages/web/src/content/docs/da/go.mdx:155-156`). No prose changes, no row reordering, no other models touched.
- The fix is internally consistent with the surrounding rows — every other entry in that table block (GLM-5, Kimi K2.5, Kimi K2.6, MiMo-V2-Pro, MiMo-V2-Omni, MiMo-V2.5-Pro) already used `/v1/chat/completions` + `@ai-sdk/openai-compatible`, so the two DeepSeek rows were the lone outliers. The contributor confirmed against the actual zen gateway behavior that DeepSeek V4 routes through the OpenAI-compatible endpoint, not the Anthropic-style messages endpoint.
- Zero code or schema impact, zero new translatable strings introduced, no `en` source file mismatch (the `en/go.mdx` row was either already correct upstream or got the same swap — worth one `git grep "deepseek-v4-pro.*messages"` post-merge to confirm nothing else still points at the wrong endpoint).

## Verdict

`merge-as-is`

## Rationale

Identical mechanical correction across 18 localized files of a documented-but-wrong endpoint, fully consistent with the surrounding table rows. Nothing to nit.
