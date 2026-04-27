# BerriAI/litellm #26651 — Fix GPT-5.5 Pro Pricing

- **PR**: [BerriAI/litellm#26651](https://github.com/BerriAI/litellm/pull/26651)
- **Head SHA**: `ea0ce944`
- **Stats**: +74/-74 across 3 files

## Summary

Halves the per-token costs for `gpt-5.5-pro`, `gpt-5.5-pro-2026-04-23`,
and the `azure/` namespaced variants of both, in
`model_prices_and_context_window.json` and the bundled
`litellm/model_prices_and_context_window_backup.json`. Prior values were
exactly double the rates published on OpenAI's pricing page, so every
billing/cost calculation against these model IDs was over-reporting by
a factor of 2x. Test updates at
`tests/test_litellm/litellm_core_utils/llm_cost_calc/test_llm_cost_calc_utils.py`
mirror the new values.

## Specific findings

- `model_prices_and_context_window.json` (and the backup at
  `litellm/model_prices_and_context_window_backup.json`) — for each of
  4 model entries, four cost fields halved consistently:
  - `cache_read_input_token_cost`: 6e-06 → 3e-06
  - `cache_read_input_token_cost_above_272k_tokens`: 1.2e-05 → 6e-06
  - `input_cost_per_token`: 6e-05 → 3e-05
  - `input_cost_per_token_above_272k_tokens`: 0.00012 → 6e-05
  - `output_cost_per_token`: 0.00036 → 0.00018
  - `output_cost_per_token_above_272k_tokens`: 0.00054 → 0.00027

  And for the non-azure entries, two more flex/batch fields:
  - `input_cost_per_token_flex`: 3e-05 → 1.5e-05
  - `input_cost_per_token_batches`: 3e-05 → 1.5e-05
  - `output_cost_per_token_flex`: 0.00018 → 9e-05
  - `output_cost_per_token_batches`: 0.00018 → 9e-05

  All ratios are exact halves — consistent with "the original entry
  doubled the per-1M-token rate when computing per-token", a common
  mistake when the source pricing page lists per-1M and the litellm
  schema expects per-1-token (off-by-1M scaling, or in this case a
  rate-card doubling).
- The new rates align with OpenAI's published pricing for gpt-5.5-pro
  ($30/1M input, $180/1M output, $3/1M cached input — confirmed in the
  PR description and reflected in the updated docstring at
  `test_llm_cost_calc_utils.py:241-242`).
- `test_llm_cost_calc_utils.py:241,275-278` — test fixtures and
  parametrized cases updated symmetrically (input cost 6e-5 → 3e-5,
  output 3.6e-4 → 1.8e-4, cached 6e-6 → 3e-6) so the test suite
  validates the new prices, not the prior wrong ones. Critically, the
  parametrized test covers all four affected model IDs (`gpt-5.5-pro`,
  `gpt-5.5-pro-2026-04-23`, `azure/gpt-5.5-pro`,
  `azure/gpt-5.5-pro-2026-04-23`) — no risk of forgetting one variant.
- Both the canonical `model_prices_and_context_window.json` and the
  bundled `litellm/model_prices_and_context_window_backup.json` are
  updated in the same PR — they must move together because the bundled
  copy is the offline fallback when the canonical can't be fetched at
  runtime.

## Nits / what's missing

- No code-side validation rule prevents a future "double the rate by
  accident" regression. A test that asserts the litellm rate matches a
  known OpenAI-published rate (perhaps via an external snapshot file
  per provider) would catch this class of bug at PR time. Out of
  scope for a same-day price-fix PR but worth filing.
- No batch/flex pricing for the `azure/*` variants in the diff —
  consistent with the prior shape (the azure entries didn't have
  `*_flex` / `*_batches` keys before either), but worth confirming
  that azure genuinely doesn't expose these endpoints for gpt-5.5-pro
  (vs the keys being missing because no one filled them in).
- The dated variant `gpt-5.5-pro-2026-04-23` is treated as a clone of
  the un-dated alias in pricing — fine if that's the actual upstream
  contract (OpenAI typically pins dated snapshots at the same price as
  the floating alias they were published under), but if there's any
  divergence in the future these will need to be updated independently.

## Verdict

**merge-as-is** — straightforward pricing correction with the right
1-to-1 test update, both files (canonical + bundled fallback) updated
together, all four affected model IDs covered. Halves a 2x
over-reporting bug in cost accounting; no regression risk because the
test suite would have failed if any of the rates were wrong relative to
the new values being asserted.
