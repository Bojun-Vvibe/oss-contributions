# Review: BerriAI/litellm#27056

- **PR:** BerriAI/litellm#27056
- **Head SHA:** `cd2f2b3add5146c91bbff6be89ea238f194347be`
- **Title:** feat(cost): add cost mapping for deepseek-v4-flash and deepseek-v4-pro
- **Author:** Dotify71

## Files touched (high-level)

- `model_prices_and_context_window.json` — adds 4 new entries: `deepseek-v4-flash`, `deepseek-v4-pro` (top-level keys, presumably alias-routed), and `deepseek/deepseek-v4-flash`, `deepseek/deepseek-v4-pro` (provider-prefixed canonical entries).

## Specific observations

- `model_prices_and_context_window.json:9924+` (`deepseek-v4-flash`, top-level): `input_cost_per_token: 1.4e-07`, `output_cost_per_token: 2.8e-07`, `cache_read_input_token_cost: 1.4e-08`, `max_input_tokens: 1_000_000`, `max_output_tokens: 8192`. Sources upstream pricing page `https://api-docs.deepseek.com/quick_start/pricing`. The 10× cache-read discount (1.4e-08 vs 1.4e-07 input) matches DeepSeek's stated cache-hit pricing.
- `model_prices_and_context_window.json:9944+` (`deepseek-v4-pro`, top-level): `input_cost_per_token: 1.74e-06`, `output_cost_per_token: 3.48e-06`, `cache_read_input_token_cost: 1.74e-07`. Same 10× cache discount ratio. 1M input window, 8192 output — consistent with V3's window. **Worth verifying** the pricing numbers against the live DeepSeek pricing page; the `source` field URL is correct, but the entry isn't dated.
- `model_prices_and_context_window.json:12501+` (`deepseek/deepseek-v4-flash`, provider-prefixed): adds `cache_creation_input_token_cost: 0.0` and `input_cost_per_token_cache_hit: 1.4e-08`, plus `supports_assistant_prefill: true`. The two cache-cost representations (`cache_read_input_token_cost` vs `input_cost_per_token_cache_hit`) coexist for backward-compat with both naming conventions in the litellm cost-tracker. Same for `deepseek/deepseek-v4-pro`.
- The top-level entries (`deepseek-v4-flash`, `deepseek-v4-pro`) lack `cache_creation_input_token_cost` and `input_cost_per_token_cache_hit` that the provider-prefixed entries have. Asymmetry is intentional (top-level entries are short-form, provider-prefixed are canonical) and matches the existing pattern used by other DeepSeek entries in the file.
- `supported_endpoints: ["/v1/chat/completions"]` — correct, DeepSeek does not expose `/v1/messages` style endpoints.
- `supports_response_schema: true` — worth double-checking against the DeepSeek API docs; V3 supports JSON mode but "structured outputs" / strict schema is API-version-dependent. If V4-flash/V4-pro do not actually enforce schema strictly, this flag will mislead callers. Low-confidence flag from a docs page that may not yet be updated.
- No `deprecation_date` / `release_date` metadata, but this is the existing convention in this file for newly added models.

## Verdict

**merge-after-nits**

## Reasoning

This is a small, well-formed cost-table addition matching the existing structure for DeepSeek entries. Two quick verifications before merge: (1) cross-check the input/output/cache-hit numbers against the live `api-docs.deepseek.com/quick_start/pricing` page (the diff has them but they're not dated, and DeepSeek has historically adjusted pricing without bumping model names), and (2) confirm `supports_response_schema: true` is actually true for v4-flash/v4-pro per the DeepSeek API reference, not just v3-style JSON mode. Both are 30-second checks against upstream docs. No code changes, lowest-risk PR class.
