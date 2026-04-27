# BerriAI/litellm PR #26589 — cost calculation for the Responses API across all providers

- **PR**: https://github.com/BerriAI/litellm/pull/26589
- **Author**: @hunterchris
- **Head SHA**: `b7b4b9b4a8814c946f692f4d1f4530cc6d6b9ab7`
- **Size**: +424 / −4
- **Files**: `litellm/llms/openai/responses/transformation.py`, `tests/test_litellm/llms/openai/responses/test_openai_responses_cost_calculation.py`

## Summary

Fixes a "cost calculation simply doesn't run" bug for `/v1/responses` across 60+ providers (per the author: 91% of requests showed `$0.00`, breaking budget enforcement and audit). The fix adds `_calculate_response_cost(model, usage_dict, response_obj)` to the OpenAI Responses transformation, which:

1. Maps Responses-API field names (`input_tokens`, `output_tokens`, `input_tokens_details`) to the standard `Usage` shape (`prompt_tokens`, `completion_tokens`, `prompt_tokens_details`).
2. Resolves the per-provider string from `self.custom_llm_provider` (handling both `LlmProviders` enum and plain string).
3. Calls `generic_cost_per_token(model, usage, custom_llm_provider, service_tier=None)` and stores the result on `response_obj._hidden_params["response_cost"]`.

Hooked into `transform_response_api_response` immediately after the `ResponsesAPIResponse` is constructed.

## Verdict: `request-changes`

The bug is real and the architectural placement is correct (the OpenAI-shaped Responses transform is the right convergence point for all 60+ providers that flow through it). Three things keep me from voting merge:

1. **`service_tier=None` is a `TODO` left in production code.** `generic_cost_per_token` will charge `flex` / `priority` / `default` rates differently on OpenAI; landing this with a hard-coded `None` will either over-bill `flex` users or under-bill `priority` users for every Responses API call. This is the *cost-correctness* PR — leaving a known cost-incorrectness `TODO` defeats the framing. The request body / `kwargs` already carry `service_tier` for completion calls; plumb it before merge.
2. **Silent `except Exception: pass`** at the bottom of `_calculate_response_cost` swallows every error to a `verbose_logger.error`. For a fix whose explicit purpose is "make cost calculation reliable", silently returning a `$0` cost on any pricing-data shape change (a new `cached_tokens` field, a new model added without pricing, a new `service_tier` value) is exactly the failure mode that produced the original bug. At minimum, increment a counter (`litellm.cost_calculation_errors`) so deployments can alert; better, fail-closed for callers that have opted into strict cost tracking.
3. **`Usage(**mapped_usage)` constructor call** drops any field that's neither `prompt_tokens`, `completion_tokens`, `total_tokens`, nor the two `_details` blocks. Responses API has emitted `cache_creation_input_tokens` / `reasoning_tokens` for some providers; if those don't survive into the `Usage` object before `generic_cost_per_token` runs, cache-write and reasoning costs will be invisible. Worth either an explicit allow-list comment or a `**{k:v for k,v in usage_dict.items() if k in USAGE_PASS_THROUGH}` that documents what's intentionally dropped.

## Specific references

- `litellm/llms/openai/responses/transformation.py:39-110` — full `_calculate_response_cost`. The mapping logic at `:55-66` is correct as far as it goes, but see issues 1 and 3 above. The provider-resolution shim at `:78-82` (`provider_val.value if isinstance(provider_val, LlmProviders) else str(provider_val)`) is defensive but suggests `self.custom_llm_provider`'s type contract is unclear in this codebase — a follow-up that pins it to one shape would prevent the next class of bug.
- `litellm/llms/openai/responses/transformation.py:99-110` — the silent-swallow:
  ```python
  except Exception as e:
      verbose_logger.error(...)
      pass
  ```
  See issue 2.
- `litellm/llms/openai/responses/transformation.py:312-315` — call site inside `transform_response_api_response`:
  ```python
  usage_dict = raw_response_json.get("usage", {})
  self._calculate_response_cost(model, usage_dict, response)
  ```
  Correct placement: after `ResponsesAPIResponse.model_construct(**raw_response_json)` (so `response._hidden_params` exists) and before `additional_headers` are stamped (so the cost is visible to header construction). Good.
- `tests/test_litellm/llms/openai/responses/test_openai_responses_cost_calculation.py` — the new test file is referenced but not in the diff snippet I read; before merge, verify it covers (a) gpt-4o family, (b) one Azure-deployed model, (c) one third-party (Together / Fireworks / Groq) where pricing data lives in `model_prices_and_context_window.json`, (d) the `cached_tokens` path, (e) `service_tier` once issue 1 is addressed.

## Nits

1. The docstring at `:38` says "Calculate cost from Responses API usage data" but the implementation also stores it; rename to `_calculate_and_store_response_cost` or update the docstring.
2. The `_calculate_response_cost` method takes `response_obj: Any` — it actually requires `response_obj` to have a mutable `_hidden_params` dict. Tighten to `response_obj: ResponsesAPIResponse` (or a `Protocol`) so the contract is enforceable.
3. `verbose_logger.debug(f"Responses API cost calculated for {model}: ...")` will fire on every successful request — at high QPS this is hot-path log spam. Move to a `cost_logger` namespace or guard on a feature flag.

## What I learned

"This bug affects 60+ providers" framings in PR descriptions are usually undersold or oversold; this one is correctly sold (the OpenAI Responses transform really is the convergence point). The interesting design question this PR raises and doesn't answer is *why* cost calculation lives in the per-provider transformation layer instead of in a shared post-response middleware. The answer ("per-provider has access to provider-specific usage shapes") is fine, but it means every new response-shaped API surface (Realtime, Assistants v3, future) has to remember to call `_calculate_response_cost` again. A `BaseTransformation.transform_response` that calls `_compute_cost(self, response)` as a Template-Method final step would prevent the next class of "we forgot to compute cost" bug.
