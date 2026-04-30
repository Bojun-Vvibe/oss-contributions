# BerriAI/litellm #26870 — fix(streaming): propagate provider cost through streaming chunk builder

- Head SHA: `a03ba7ba72d77a2705266422932dbc3ea7cb2142`
- Files: `litellm/litellm_core_utils/streaming_chunk_builder_utils.py`,
  `litellm/main.py`,
  `litellm/types/litellm_core_utils/streaming_chunk_builder_utils.py`,
  `tests/test_litellm/llms/openrouter/chat/test_openrouter_chat_transformation.py`
- Size: +77 / -0
- Closes #16021

## What it does

OpenRouter (and other providers using its convention) emits `usage.cost` in
the final streaming chunk. `chunk_parser` was already preserving this on each
`ModelResponseStream`, but the four-step pipeline that rebuilds a single
non-streaming-shaped response from streamed chunks dropped it on the floor.
Specifically:

1. `_calculate_usage_per_chunk` did not extract `cost` while iterating
   `usage_chunk` objects — so the per-chunk accumulator never even saw it.
2. `calculate_usage` had no path to put `cost` onto the returned `Usage`.
3. `stream_chunk_builder` had no bridge from `usage.cost` to the
   `_hidden_params["additional_headers"]["llm_provider-x-litellm-response-cost"]`
   key that `response_cost_calculator` reads in the non-streaming path.

End result: streaming users on OpenRouter (and any provider whose cost is
authoritative from the upstream rather than computed locally) silently lost
cost telemetry. The non-streaming path already worked correctly, so the bug
was purely in the stream-rebuild seam.

## The fix

Four small, well-targeted changes:

1. `streaming_chunk_builder_utils.py:547` — accumulator init `cost: Optional[float] = None`.
2. `streaming_chunk_builder_utils.py:593-597` — extract via
   `getattr(usage_chunk, "cost", None)` with a `isinstance(usage_chunk, dict)`
   fallback for plain-dict usage chunks. The `if _cost is not None: cost = _cost`
   guard correctly preserves `cost=0.0` (a legitimate value for free-tier
   models or zero-cost cached responses) while letting later non-cost-bearing
   chunks not clobber an earlier value. **Last-write-wins on cost** is the
   right semantic here since OpenRouter only ever emits cost in the final
   usage chunk.
3. `streaming_chunk_builder_utils.py:723-724` — surface to `returned_usage.cost`.
4. `litellm/main.py:7465-7470` and `litellm/main.py:7649-7654` — bridge
   `usage.cost` into `_hidden_params["additional_headers"]
   ["llm_provider-x-litellm-response-cost"]` at both stream rebuild paths
   (the choices-bearing and the choices-less variants).
5. Type extension at `types/.../streaming_chunk_builder_utils.py:17` adding
   `cost: Optional[float]` to `UsagePerChunk` keeps the TypedDict honest.

## Test coverage

`test_openrouter_streaming_cost_propagated_to_final_response` at
`tests/.../test_openrouter_chat_transformation.py:531-585` is end-to-end and
right-shaped: chunk1 has only `delta.content`, chunk2 carries
`usage.cost = 0.00042`, both go through the real
`OpenRouterChatCompletionStreamingHandler.chunk_parser`, then into the real
`stream_chunk_builder`. The assertion checks both `final_response.usage.cost
== 0.00042` *and* the `_hidden_params["additional_headers"]
["llm_provider-x-litellm-response-cost"] == 0.00042` — i.e. it locks both
the data path and the downstream consumer's lookup key.

## Why merge-as-is

Bug-for-bug additive: nothing changes for providers that don't emit
`usage.cost`. The two `_provider_response_cost = getattr(usage, "cost",
None)` blocks at `main.py:7465` and `main.py:7649` are duplicated, but
the duplication mirrors an existing pattern in the same file (the two
stream-rebuild branches each handle their own `setattr(response, "usage",
usage)` already). The dict/object dual-path extraction at
`streaming_chunk_builder_utils.py:593-595` matches the surrounding code's
existing handling of `usage_chunk` polymorphism. Test is end-to-end and
references real OpenRouter chunk shapes.

Verdict: merge-as-is
