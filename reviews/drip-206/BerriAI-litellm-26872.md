# BerriAI/litellm #26872 — fix: apply cache_read_input_token_cost in custom pricing path

- Head SHA: `f4882f562d54d7984fe43125b4f9bed99df512f9`
- Files: `litellm/cost_calculator.py`, `tests/test_litellm/test_cost_calculator.py`
- Size: +71 / -1
- Closes #26807

## What it does

Real correctness bug. The model-cost-map cost path correctly applies
`cache_read_input_token_cost` to cached prompt tokens, but the
`_cost_per_token_custom_pricing_helper` early-return branch at
`cost_calculator.py:175-188` (pre-fix) charged 100% of `prompt_tokens` at the
flat `input_cost_per_token`, ignoring `cache_read_input_token_cost` even
when the caller explicitly supplied it in `custom_cost_per_token`. For users
on custom pricing with high cache-hit rates this silently over-bills by
roughly `cached_tokens * (input_cost - cache_read_cost)` per request.

The fix adds an `usage_object: Optional[Usage] = None` parameter to the
helper (`cost_calculator.py:178`), and at lines 184-194 splits the input
cost in two when both `cache_read_input_token_cost` is present in the
`CostPerToken` dict *and* `usage_object` is provided:

```python
details = _parse_prompt_tokens_details(usage_object)
cached_tokens = details["cache_hit_tokens"]
non_cached_tokens = prompt_tokens - cached_tokens
input_cost = (
    custom_cost_per_token["input_cost_per_token"] * non_cached_tokens
    + cache_read_cost * cached_tokens
)
```

The plumbing change is a single `usage_object=usage_block` kwarg added at
the helper's call site (`cost_calculator.py:343`), which keeps the public
`cost_per_token(...)` signature and behaviour unchanged for callers that
don't supply `cache_read_input_token_cost`.

## Test coverage

The new `test_custom_pricing_applies_cache_read_token_cost` at
`tests/test_litellm/test_cost_calculator.py:2001-2056` constructs a
`Usage(prompt_tokens=6074, completion_tokens=285, prompt_tokens_details=
PromptTokensDetailsWrapper(cached_tokens=3456))` and asserts
`pytest.approx(cost, rel=1e-6) == non_cached_tokens * input_cost +
cached_tokens * cache_read_cost + completion_tokens * output_cost`. The
exact numbers in the docstring tie back to issue #26807, which makes the
regression intent unambiguous.

## What works

- Backward compatible: the new branch only activates when *both* fields
  are present; the original `prompt_tokens * input_cost_per_token` line
  remains the fallback at `cost_calculator.py:194`.
- The `# type: ignore` at `cost_calculator.py:184` is the right call —
  `CostPerToken` is a `TypedDict`-like with optional cache fields and
  pyright/mypy don't always narrow `.get()` cleanly through it.
- Reuses `_parse_prompt_tokens_details(usage_object)` rather than
  re-walking `prompt_tokens_details.cached_tokens` manually, keeping the
  cache-hit definition consistent with the model-cost-map path.

## Concerns

1. **No write-cache symmetry.** `custom_cost_per_token` also supports
   `cache_creation_input_token_cost`. The model-cost-map path bills it,
   this fix doesn't. A user on custom pricing with both read and write
   cache costs will still be over-billed on the write side. Worth
   adding the symmetric branch in this PR or filing a follow-up.
2. **`output_cost` in `_cost_per_token_custom_pricing_helper`** still
   ignores `output_cost_per_token` cache breakdowns (e.g. reasoning
   tokens). Out of scope here, but the fact that the custom-pricing
   helper is a divergent reimplementation of the cache-aware logic is
   the underlying bug — this PR fixes one symptom, not the root cause.
3. **`usage_block` propagation.** Worth a one-line comment at
   `cost_calculator.py:343` explaining why `usage_block` is now
   threaded through (so the next refactor doesn't drop it as unused).

Verdict: merge-after-nits
