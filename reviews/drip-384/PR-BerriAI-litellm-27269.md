# BerriAI/litellm#27269 — fix(responses): handle response.function_call_arguments.done for models without deltas

- **Head SHA**: `f831f3e898e574ada25350101a38de90d7328522`
- **Stats**: +119 / -0, 2 files (Fixes #27144)

## Summary

OpenAI's Responses-API streaming protocol normally emits incremental `response.function_call_arguments.delta` events followed by a terminal `response.function_call_arguments.done` event. But some upstream models (notably the ChatGPT Spark route) skip the deltas entirely and deliver complete tool-call arguments only in the `.done` event. The previous translation layer only consumed `.delta` events to reconstruct the tool-call argument string and treated `.done` as an empty pass-through, so for these models the tool's `function.arguments` arrived as `""` — which downstream tool-dispatch code then rejected, surfacing as "tool produced no arguments" or a JSON parse error on empty input.

## Specific citations

- `litellm/completion_extras/litellm_responses_transformation/transformation.py:1066-1068`: adds `self._tool_call_has_deltas: set = set()` to `__init__`. Per-output-index tracking (not per-call) is correct because a single response can have multiple parallel tool calls each at distinct `output_index` values.
- `:1389-1397`: classifies the inbound chunk by `event_type`. For `response.function_call_arguments.delta` it just records the index in the set; for `.done` it checks whether deltas were seen.
- `:1398-1430`: the load-bearing branch — when `.done` arrives without preceding deltas, synthesizes a `ModelResponseStream` with a single `StreamingChoices` containing one `ChatCompletionToolCallChunk` with `id=None`, `index=output_index`, `type="function"`, `function=ChatCompletionToolCallFunctionChunk(name=None, arguments=full_args)`. Correct shape — `name=None` is right because the function name was already emitted in an earlier `response.output_item.added` event for this index.
- `:1431-1434`: when deltas WERE received, the comment correctly explains that fall-through to the static method's pass-through behavior is desired (args are already in the stream).
- `tests/test_litellm/llms/chatgpt/responses/test_chatgpt_responses_transformation.py:200-269`: three tests via `_make_iterator()` helper that bypasses the constructor and directly seeds `_tool_call_has_deltas`: (a) `test_done_only_emits_arguments` asserts the `.done`-only path emits arguments at the right index, (b) `test_done_after_delta_does_not_duplicate` feeds delta-then-done and asserts no double-emission, (c) `test_done_without_arguments_is_noop` covers empty-args-done.

## Verdict

**merge-after-nits**

## Rationale

Correct fix for a real protocol-shape mismatch. The per-`output_index` tracking is the right granularity (parallel tool calls work) and the synthesized `ChatCompletionToolCallChunk` shape matches what the rest of the pipeline expects. The test trio covers the three meaningful axes (delta-absent, delta-present, args-empty) and the `_make_iterator()` helper that bypasses the constructor is a pragmatic test-isolation technique for a class that needs a real stream. Three nits: (1) the `_tool_call_has_deltas: set` annotation should be `Set[int]` with proper typing import — the bare `set` is fine at runtime but loses the index-type contract. (2) The "if deltas WERE received, fall through to the static method" comment at `:1431-1434` describes intent but the actual fall-through is implicit — extracting an explicit `# else: deltas_seen, fall through to static method handling` comment immediately before the `verbose_logger.debug(...)` line at `:1437` would prevent a future refactor from accidentally inserting an `else: return` and breaking this. (3) The set is never cleared — if a single iterator is reused across multiple responses (don't think it is, but worth verifying), it could leak `output_index` entries across responses; a `reset_tool_call_state()` hook called from a response-boundary event would future-proof. None block merge.
