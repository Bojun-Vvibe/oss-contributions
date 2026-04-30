# BerriAI/litellm#26870 — fix(streaming): propagate provider cost through streaming chunk builder

- URL: https://github.com/BerriAI/litellm/pull/26870
- Head SHA: `b48610f35f2559246c5d5120e6f265fd574cfbbf`
- Size: +77 / -0

## Summary

Closes a parity gap for OpenRouter (and any provider that emits `usage.cost` on the final streaming chunk) where the cost field was being silently dropped during `stream_chunk_builder` reconstruction. Adds `cost: Optional[float]` to the `UsagePerChunk` TypedDict (`streaming_chunk_builder_utils.py:16` types file), threads it through `_calculate_usage_per_chunk` accumulator (`streaming_chunk_builder_utils.py:546-621`), and propagates it onto `response._hidden_params["additional_headers"]["llm_provider-x-litellm-response-cost"]` at both reassembly sites in `main.py:7462-7468` and `:7646-7652` so `response_cost_calculator` finds it in the same key the non-streaming path uses.

## Observations

- `streaming_chunk_builder_utils.py:590-594` — the cost-extraction path handles both attribute access (`getattr(usage_chunk, "cost", None)`) and the `isinstance(usage_chunk, dict)` fallback. The accumulator overwrites `cost` on each chunk that has a non-`None` value (last-non-`None`-wins), which matches OpenRouter's contract of emitting the final cost on the terminal chunk. If a future provider streamed incremental cost deltas, this would silently take only the last one — worth a comment naming the contract.
- `main.py:7462-7468` and `:7646-7652` are two near-identical copies of the same `_provider_response_cost = getattr(usage, "cost", None) ... response._hidden_params.setdefault("additional_headers", {})["llm_provider-x-litellm-response-cost"] = _provider_response_cost` block. Both paths need the change (the function has two distinct reassembly arms — the cached/short-circuit path and the full-rebuild path) but the duplication should at minimum be a private helper `_propagate_streaming_response_cost(response, usage)` so the next addition (e.g. a per-tier cost field) doesn't need to land in two places and risk one drifting.
- `setdefault("additional_headers", {})` is the correct shape for the handoff to `response_cost_calculator` — verified against the existing non-streaming call site that uses the same key. The `llm_provider-x-` prefix matches the convention for provider-supplied cost passthrough.
- Test at `test_openrouter_chat_transformation.py:528-580` (new `test_openrouter_streaming_cost_propagated_to_final_response`) exercises the end-to-end shape: 2 chunks (one delta, one terminal-with-`usage.cost=0.00042`), then `stream_chunk_builder(...)` and asserts both `final_response.usage.cost == 0.00042` AND the `_hidden_params` header lands at the documented key. This is the right axis-covering shape — a regression that strips either the `Usage.cost` field OR the header propagation will fail loudly. Missing: a negative test asserting that absent `usage.cost` does NOT inject a `None` header value, and a test for the cached/short-circuit reassembly arm at `main.py:7462`.

## Verdict

**merge-after-nits**

## Nits

- Extract the duplicated `_propagate_streaming_response_cost(response, usage)` helper to remove the two-site copy in `main.py`.
- Add a negative test (`usage.cost=None` ⇒ no `llm_provider-x-litellm-response-cost` key in `_hidden_params`).
- Add a one-line comment in `_calculate_usage_per_chunk` documenting the last-non-`None`-wins contract on the `cost` field.
