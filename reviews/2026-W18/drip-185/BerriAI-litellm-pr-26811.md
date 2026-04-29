---
pr: BerriAI/litellm#26811
sha: fe9504d9835ac66dcd98db292ab80eb28e7ffa9b
verdict: merge-after-nits
reviewed_at: 2026-04-30T00:00:00Z
---

# fix: price cached tokens in custom cost calculator

URL: https://github.com/BerriAI/litellm/pull/26811
Files: `litellm/cost_calculator.py` (+ logging breakdown plumbing)
Diff: 575+/59-

## Context

When users supplied `custom_cost_per_token` (overriding model pricing
for non-model-card-listed deployments), the `_cost_per_token_custom_pricing_helper`
at `cost_calculator.py:182` only applied
`input_cost_per_token * prompt_tokens` and
`output_cost_per_token * completion_tokens`, silently ignoring any
`cache_read_input_token_cost` / `cache_creation_input_token_cost` keys
the caller had set. Result: every Anthropic-via-passthrough or Bedrock-via-
custom-pricing setup that was supposed to bill cached tokens at the
discounted rate was actually billing them at the full input rate.

## What's good

- The detection helper `_custom_cost_per_token_has_cache_pricing`
  at `:213-221` checks for `cache_read_input_token_cost*` /
  `cache_creation_input_token_cost*` prefixes, correctly capturing
  both the basic keys and the time-tier suffixes
  (`_above_1hr`, `_above_200k_tokens`) that the recently-added
  Bedrock 1-hour cache pricing relies on.
- The dispatch at `:191-203` is clean: when cache-pricing keys are
  present *and* a `usage_object` is available, it routes through
  `generic_cost_per_token_from_model_info` rather than the simple
  prompt/completion math — reusing the already-tested pricing engine
  instead of duplicating logic.
- `_get_model_info_from_custom_cost_per_token` at `:225-242`
  constructs a synthetic `ModelInfo` with the required core fields
  then `model_info.update(cast(dict, custom_cost_per_token))` to
  layer the cache-pricing keys on top — atomic, type-safe, and the
  `key="custom_cost_per_token"` sentinel makes downstream
  identification trivial.
- `_get_cache_costs_for_breakdown` at `:1065+` plumbs the cached
  read/write token counts into the standard logging breakdown
  fields (`cache_hit_tokens`, `cache_creation_input_tokens`),
  including the defensive cross-check at `:1080` where top-level
  `cache_read_input_tokens` is used as a fallback when
  `prompt_tokens_details.cache_hit_tokens <= 0`.
- Backwards-compat preserved: callers without cache-pricing keys
  still hit the original prompt/completion branch unchanged, so
  existing custom-cost-per-token integrations don't see a behaviour
  change.

## Nits

- The `usage_object is None` check at `:191` short-circuits to the
  legacy math even when cache-pricing keys are present — that's
  *probably* the safe choice (no usage = no cache breakdown to
  apply) but a one-line `debug_print_logger.warn(...)` noting
  "custom cache pricing configured but no usage object provided,
  falling back to flat input/output rates" would make the
  silent-fallback case observable.
- `_get_model_info_from_custom_cost_per_token` at `:230` hardcodes
  `mode="chat"` and `max_*=None` — fine for cost calculation but
  means the synthetic ModelInfo can't be reused for context-window
  enforcement or mode-dependent routing. Add a comment that this
  is intentionally cost-only.
- The `cast(dict, custom_cost_per_token)` at `:218,242` is needed
  because `CostPerToken` is a TypedDict — consider exporting an
  iterable-keys helper on `CostPerToken` itself so the cast leaks
  in fewer places.
- No test in this diff exercising the new code path. A parametric
  test at the cost-calculator layer with (a) cache-read-only
  pricing, (b) cache-write-only pricing, (c) both with the
  `_above_1hr` tier, (d) `usage_object=None` fallback, (e) callers
  with cache-pricing keys but `cache_read_input_tokens=0` (the
  non-hit case), would lock the contract.
- The PR description should call out that `_cost_per_token_custom_pricing_helper`
  signature gained three parameters (`usage_object`, `custom_llm_provider`,
  `service_tier`) — every existing caller needs to be audited to
  ensure they're now passing usage when available, or the behaviour
  fix won't activate for them.
- CHANGELOG entry: this is a billing-correctness fix that will
  retroactively change cost numbers for some custom-pricing
  deployments. Operators reconciling spend logs need to know.

## Verdict reasoning

Real correctness fix that closes a silent under-billing /
over-billing gap for custom-pricing deployments using cache
discounts. Routes through the existing tested pricing engine
rather than duplicating logic, and the synthetic-ModelInfo
construction is the right shape. Nits center on test absence (the
chief blocker risk on a billing change), the silent fallback when
`usage_object` is None, and downstream caller audit. None
fundamentally blocking; cleanly landable after a parametric
regression test is added.
