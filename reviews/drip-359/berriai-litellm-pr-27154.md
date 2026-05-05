# BerriAI/litellm PR #27154 — feat(xai): add grok-4.3 and grok-4.3-latest to model_prices

- Repo: `BerriAI/litellm`
- PR: #27154
- Head SHA: `1c31e2ce3db530847d56472faf2352b4a27fab0f`
- Author: `ishaan-berri`
- Updated: 2026-05-05T01:47:39Z
- Verdict: **merge-as-is**

## What it does

Adds two new entries to `model_prices_and_context_window.json:34742-34784` — `xai/grok-4.3` and `xai/grok-4.3-latest` — both with identical values:

| field | value |
|---|---|
| `litellm_provider` | `xai` |
| `mode` | `chat` |
| `max_input_tokens` / `max_output_tokens` / `max_tokens` | `1000000` |
| `input_cost_per_token` | `1.25e-06` ($1.25 / M tokens) |
| `output_cost_per_token` | `2.5e-06` ($2.50 / M tokens) |
| `input_cost_per_token_above_200k_tokens` | `2.5e-06` (2× tier above 200k) |
| `output_cost_per_token_above_200k_tokens` | `5e-06` (2× tier above 200k) |
| `cache_read_input_token_cost` | `2e-07` |
| `cache_read_input_token_cost_above_200k_tokens` | `4e-07` |
| `supports_function_calling` / `supports_prompt_caching` / `supports_reasoning` / `supports_response_schema` / `supports_tool_choice` / `supports_vision` / `supports_web_search` | all `true` |
| `source` | `https://docs.x.ai/docs/models/grok-4.3` |

## Strengths

- **Correct schema.** Both entries follow the established xai pattern — `supports_*` flags, `cache_read_input_token_cost*`, the `*_above_200k_tokens` tier doubling, and `source` URL — and slot in alphabetically right above `xai/grok-beta` at line 34784, matching neighbor conventions.
- **`-latest` alias is symmetric.** Defining `xai/grok-4.3-latest` with the same numbers as `xai/grok-4.3` is the standard pattern for "rolling alias to the current minor"; consumers who pin `-latest` get the same cost basis as the dated tag, and when a `grok-4.3.1` patch ships only the `-latest` row needs to be edited.
- **Above-200k tier doubling is internally consistent.** Input doubles ($1.25 → $2.50), output doubles ($2.50 → $5.00), cache-read doubles ($0.20 → $0.40). Same shape as the existing tiered xai/grok entries.
- **`max_input_tokens == max_output_tokens == max_tokens == 1_000_000`.** Matches xAI's published 1M context window for the 4.x line.

## Nits

1. **Source URL not yet verifiable.** `https://docs.x.ai/docs/models/grok-4.3` resolves the same shape as other xai model pages, but no archive snapshot is linked in the PR body. For a pricing PR this isn't blocking — the maintainers presumably have access to the xAI partner channel — but a screenshot or `web.archive.org` snapshot in the PR description would help downstream auditors who don't trust live URLs.
2. **No test entry.** `tests/local_testing/test_get_model_info.py` (or wherever the model-info smoke tests live) typically gets a sanity check that newly-added entries parse and return the expected `mode` / `litellm_provider` / `max_tokens`. A two-line addition would catch a future JSON syntax break in the same PR window.
3. **`supports_reasoning: true`** — grok-4.x is the reasoning line so this is correct, but if the API doesn't yet expose a reasoning-effort knob to litellm clients, callers who set `reasoning_effort=...` may get a silent provider-level ignore. Worth a one-line note in the PR body about whether `reasoning_effort` is wired in the existing xai handler.

## Verdict
**merge-as-is** — pure pricing-catalog data add, mirrors the existing xai model entries exactly, and the tier-doubling math is internally consistent. The nits above are nice-to-haves, not blockers; pricing PRs in litellm routinely land at this size and shape.
