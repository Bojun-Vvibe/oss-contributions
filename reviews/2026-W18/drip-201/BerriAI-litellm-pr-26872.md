# BerriAI/litellm #26872 — fix: apply cache_read_input_token_cost in custom pricing path

- **Author:** Christian-Sidak
- **SHA:** `f4882f5`
- **State:** OPEN
- **Size:** +71 / -1 across 2 files (`litellm/cost_calculator.py`,
  `tests/test_litellm/test_cost_calculator.py`)
- **Verdict:** `merge-as-is`

## Summary

Closes a real cost-calculation bug where `custom_cost_per_token` callers passing
`{"input_cost_per_token", "output_cost_per_token", "cache_read_input_token_cost"}`
were being charged at the regular `input_cost_per_token` rate for cached tokens —
the cache-read field was silently ignored in the custom pricing path. Fix at
`litellm/cost_calculator.py:178-196`: extends `_cost_per_token_custom_pricing_helper`
to accept an optional `usage_object: Optional[Usage] = None` keyword, parses
`cached_tokens` via the existing `_parse_prompt_tokens_details(usage_object)` helper,
and splits prompt billing into `non_cached_tokens * input_cost + cached_tokens *
cache_read_cost` when both `cache_read_cost` and `usage_object` are present. The
caller at `cost_per_token` `:343` is updated to pass `usage_block` through. Backfilled
with a 57-line regression at `tests/test_litellm/test_cost_calculator.py:1999+` named
`test_custom_pricing_applies_cache_read_token_cost` linked to issue #26807, locking
the exact split arithmetic on `prompt_tokens=6074, cached_tokens=3456,
completion_tokens=285`.

## Reasoning

The bug is real and money-affecting — a customer using custom pricing with a 50%+
cache-hit ratio was being overcharged by the ratio of `input_cost_per_token /
cache_read_input_token_cost` on the cached portion (the regression test's chosen
ratio is `2.5e-6 / 2.5e-7 = 10×`, so the test scenario was being over-billed by ~9×
the cached-token cost component, ~$0.0086 per request in the test fixture). The fix
correctly preserves the legacy "no cache_read_cost OR no usage_object" path so any
existing caller that didn't supply a usage object continues to get the old
behavior. The new `usage_object: Optional[Usage] = None` keyword at `:178` is a
backward-compatible signature extension. The dict-access pattern
`custom_cost_per_token.get("cache_read_input_token_cost")` at `:185` matches the
duck-typed `CostPerToken` TypedDict shape used elsewhere in the file. The regression
asserts the exact expected total at `:2049-2056` which will fail loudly on any future
arithmetic drift. No banned-string surface, no security implications, single
load-bearing call site updated correctly.
