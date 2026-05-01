# BerriAI/litellm#26971 — feat: add Cortecs AI as named OpenAI-compatible provider

- **PR**: https://github.com/BerriAI/litellm/pull/26971
- **Head SHA**: `19da4686f24ef84ef2947644884f596f90df18b7`
- **Size**: +38 / -0, 3 files
- **Verdict**: **merge-after-nits**

## Context

Adds Cortecs AI as a named OpenAI-compatible provider. Three JSON-only edits:
- `litellm/llms/openai_like/providers.json` (provider registry)
- `provider_endpoints_support.json` (endpoint capability matrix)
- `litellm/provider_endpoints_support_backup.json` (backup of the same)

## What's right

**`providers.json:108-112` — minimal registry entry.**

```json
"cortecs": {
  "base_url": "https://api.cortecs.ai/v1",
  "api_key_env": "CORTECS_API_KEY"
}
```

Follows the existing OpenAI-compatible-provider shape (compare to the `aihubmix` entry directly above at `:103-107`). Notably *omits* the `api_base_env` field that aihubmix carries — for a fixed `base_url` provider that's correct, since users can already override via the standard `api_base` parameter on `completion()`.

**`provider_endpoints_support.json:512-528` and `..._backup.json:477-493` — endpoint matrix entries are kept in sync.**

Both files get the same 17-line block declaring `chat_completions: true` and every other endpoint (`messages`, `responses`, `embeddings`, `image_generations`, `audio_transcriptions`, `audio_speech`, `moderations`, `batches`, `rerank`, `a2a`) `false`. The narrow capability declaration matches what an OpenAI-compatible chat-only provider should expose, and the dual-file mirror discipline matches the surrounding pattern (every other entry in `_backup.json` is a verbatim copy of the canonical file's entry).

**Insertion point is alphabetical within the existing block.** `cortecs` lands between `clarifai` (line 494 in canonical) — wait, actually it lands *before* `clarifai` in the diff, not after. The diff shows `cortecs` inserted at `:512`/`:477` and the existing `clarifai` block immediately following at `:529`/`:494`. That is *not* alphabetically correct — `clarifai` < `cortecs` so cortecs should follow clarifai. Worth a one-line reorder.

## Risks / nits

- **Alphabetical ordering nit.** As above: cortecs is inserted *before* clarifai in both endpoint-support JSONs (visible in the diff context where `"clarifai":` follows the new `"cortecs":` block). The `providers.json` ordering at the file's tail is also worth verifying against neighbors — the diff shows it appended after `aihubmix` which is fine if those two are at the file's end, but skim the full file before merge.
- **No corresponding docs page.** The `url` field at `:514` and `:479` points to `https://docs.litellm.ai/docs/providers/cortecs` — confirm that doc page is being added in a sibling PR or that the link gracefully 404s without breaking the provider listing UI on docs.litellm.ai. The PR description doesn't reference an accompanying docs PR.
- **No model entries in `model_prices_and_context_window.json`.** The PR registers the *provider* but lists no specific Cortecs models with their token prices, so cost reporting / metadata for `cortecs/<model-name>` calls will fall back to defaults. Other named-provider PRs in the same area (e.g. #26970 adding Venice models) typically bundle pricing — worth confirming that's a deliberate phase-1 / "register provider, prices in follow-up" choice and noting it in the PR body.
- **No test arm.** A one-line addition to the OpenAI-like provider test suite (`tests/llm_translation/test_openai_like_chat_completion.py` or similar) asserting `litellm.completion(model="cortecs/test-model", ...)` resolves to `base_url="https://api.cortecs.ai/v1"` and reads `CORTECS_API_KEY` from env would pin the regression surface for free.
- **API base URL trailing slash.** `https://api.cortecs.ai/v1` (no trailing slash) — confirm the OpenAI-compatible HTTP client appends `/chat/completions` cleanly without a double-slash or missing-slash bug. Most providers in the file use the same shape so this is likely fine, but worth a single live curl during review.
- **GDPR/EU jurisdiction note in PR body** ("Popular in Europe (GDPR-native)") doesn't translate to any code-level routing or data-residency guarantee — purely advisory in this PR. Fine, just don't oversell in any future docs.

## Verdict

**merge-after-nits.** Mechanical provider-registration PR, follows the established pattern. Fix the alphabetical ordering before merge (one move, no behavior change), confirm the docs-page is being added separately, and consider the model-pricing follow-up. Code itself is safe to land.
