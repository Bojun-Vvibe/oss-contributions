# BerriAI/litellm PR #26848 — adds deepseek-v4 pricing

- PR: https://github.com/BerriAI/litellm/pull/26848
- Head SHA: `9bdc5accbf414c37e445f845d6cd40a420e25af9`
- Files touched: 1 (`model_prices_and_context_window.json` +15/-0).

## Specific citations

- One new model entry inserted at `model_prices_and_context_window.json:12471-12485` keyed `"deepseek/deepseek-v4-pro"`. Fields:
  - `max_tokens: 384000`, `max_input_tokens: 1000000`, `max_output_tokens: 384000`
  - `input_cost_per_token: 4.35e-07` (i.e., $0.435/1M), `output_cost_per_token: 8.7e-07` ($0.87/1M)
  - `cache_read_input_token_cost: 3.625e-09` ($0.003625/1M)
  - `litellm_provider: "deepseek"`, `mode: "chat"`
  - Capability flags: `supports_assistant_prefill`, `supports_function_calling`, `supports_prompt_caching`, `supports_reasoning`, `supports_tool_choice` all `true`
- The entry slots in immediately before the existing `deepseek.v3-v1:0` Bedrock entry at `:12486-`, keeping alphabetic-ish grouping by provider prefix.

## Verdict: merge-after-nits

## Concerns / nits

1. **No fixture/snapshot update verifiable in this diff slice.** Other model-pricing additions in this repo (per `drip-34` review of #26485) flagged the same gap: a model registry entry with no model-cost fixture row will silently log $0 spend on first use until a separate PR backfills the fixture. Worth either bundling the fixture row in this PR or linking the follow-up PR in the description.
2. **`cache_read_input_token_cost: 3.625e-09`** is suspiciously small — $0.003625 per 1M cache-read tokens, vs `input_cost_per_token: 4.35e-07` ($0.435/1M) — i.e., cache reads are ~120x cheaper than fresh input. DeepSeek's published cache-hit pricing is normally ~10% of input (ratio 10x, not 120x). Either the upstream provider has aggressively undercut their own cache pricing for v4-pro, or there's a unit error (off by a factor of 10) in the constant. Worth a doc-link in the PR description to the DeepSeek pricing page so reviewers can verify the 120x ratio is intentional.
3. **No `cache_creation_input_token_cost`** entry. DeepSeek's V4 family typically charges for cache writes too. If their V4-pro tier is genuinely free-on-write, that's worth asserting in the PR body. Otherwise the price model is incomplete and downstream cost reporting will undercount.
4. **No release-date or `deprecation_date` metadata.** Several other deepseek entries in the file carry `release_date` / `deprecation_date`. Including those here would help downstream model selectors that prefer latest-available models.
5. **No matching alias entry.** The PR adds `deepseek/deepseek-v4-pro` only. Other entries (e.g., the openrouter-routed and bedrock-routed variants) tend to land in the same PR. If users will route through `openrouter/deepseek/deepseek-v4-pro` or `bedrock/deepseek-v4-pro`, those entries are still missing and will fall back to default $0 cost tracking. Worth a follow-up issue at minimum.
