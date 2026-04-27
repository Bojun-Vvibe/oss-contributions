# PR #26612 — feat(pricing): add Snowflake Cortex REST API model pricing

- **Repo**: BerriAI/litellm
- **PR**: #26612
- **Head SHA**: `ebeec24d`
- **Author**: Nav25oct
- **Size**: +247 / -37 across 1 file
- **URL**: https://github.com/BerriAI/litellm/pull/26612
- **Verdict**: **merge-after-nits**

## Summary

Updates `model_prices_and_context_window.json` to reflect the
current public Snowflake Cortex REST API surface: bumps stale
context windows on the Anthropic-passthrough Claude entries (e.g.
`snowflake/claude-3-5-sonnet` from 18k → 200k input, 8k → 16k
output) and on the larger Llama variants (32k/128k → 128k input,
8k → 16k output), adds prices that were previously missing
(`input_cost_per_token`, `output_cost_per_token`,
`cache_read_input_token_cost` where applicable), and toggles the
correct capability flags (`supports_function_calling`,
`supports_vision`, `supports_prompt_caching`,
`supports_system_messages`, `supports_response_schema`) on the
Claude row. Adds a new entry shape for several llama variants
that previously had no pricing data at all, and tags
`snowflake/llama3.3-70b` with the same `supports_*` set as its
siblings.

## Specific changes

- `model_prices_and_context_window.json:28876-28890` —
  `snowflake/claude-3-5-sonnet` gets the full Anthropic feature
  matrix and per-token pricing matching Anthropic's published
  rates passed through Snowflake Cortex (`3e-6` in / `15e-6` out
  / `3e-7` cache-read), matching Anthropic's published rate sheet
  for the underlying model.
- `model_prices_and_context_window.json:28891-28906` —
  `snowflake/deepseek-r1` context bumped from 32k → 128k input
  (matches DeepSeek-R1 native context), `max_output_tokens`
  bumped to 16384, prices added, `supports_system_messages: true`
  added.
- `model_prices_and_context_window.json:28944-28998` — the
  `llama3.1-405b`, `llama3.1-70b`, `llama3.1-8b` rows get
  per-token pricing that matches Snowflake Cortex's published
  rates and `supports_function_calling`/`supports_system_messages`
  flags lit up.
- `model_prices_and_context_window.json:28998-29010` —
  `snowflake/llama3.3-70b` row reformatted (the existing entry
  was missing `litellm_provider` repositioning); content-equivalent
  but the diff position changes a few keys' order.

## Risks

Three review-level concerns, all easily addressed before merge:

1. **JSON formatting drift**: the diff has multiple places where
   the entry indentation changes from 4 spaces to 6 spaces (look
   at `snowflake/claude-3-5-sonnet`, `snowflake/deepseek-r1`,
   `snowflake/llama3.3-70b` opening braces — they're now
   double-indented vs the surrounding entries). This file is
   diff-sensitive because every entry change shows up as a
   conflict on parallel PRs. A `prettier --write` or matching
   the existing 4-space indent fixes it.

2. **No test pinning the new pricing**: litellm has a pattern of
   pricing-invariant tests that load
   `model_prices_and_context_window.json` and assert specific
   model entries match expected fields. Bumping `claude-3-5-sonnet`
   from 18k → 200k input is a >10× change visible in cost
   estimation and routing decisions; a test that loads the row
   and asserts `max_input_tokens == 200000` etc. would catch a
   future merge accidentally reverting it.

3. **`cache_read_input_token_cost` only on Claude**: the other
   Snowflake-Anthropic-passthrough models (if any are added in
   the future) would silently miss this and undercount cost.
   The PR description should call out that prompt caching pricing
   is currently Claude-only on Snowflake's side and the row reflects
   that, so future reviewers know not to copy-paste the field
   onto other providers.

## Verdict rationale

The pricing data itself looks right against Snowflake Cortex's
public rate page and the capability-flag additions match the
underlying model contracts. But the indentation drift will hurt
review of subsequent pricing PRs and there's no regression test
pinning the new numbers. Merge after the formatting is normalized
to 4-space and a basic `assert row["max_input_tokens"] == 200000`
test is added for the headline Claude row.
