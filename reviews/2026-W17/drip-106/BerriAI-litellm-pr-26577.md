# BerriAI/litellm PR #26577 — fix(proxy): skip redundant tiktoken recount when provider supplies reasoning_tokens

- **Link**: https://github.com/BerriAI/litellm/pull/26577
- **Head SHA**: `28c5f32ca9c54770452d88db1fed1d6cef0ff164`
- **Author**: dschulmeist
- **Size**: +380 / -11 across 8 files
- **Verdict**: `merge-after-nits`

## Files changed
- `litellm/litellm_core_utils/streaming_chunk_builder_utils.py:509-555` — adds `chunks_have_reasoning_tokens(self, chunks)` which scans all streaming chunks for any usage chunk where `completion_tokens_details.reasoning_tokens is not None`. Defensive about chunk shape: handles both dict and `ModelResponse`-like objects, both for the outer chunk and for the nested `usage` and `completion_tokens_details`. Falls back to `_hidden_params['usage']` for chunks where `usage` was hoisted out.
- `litellm/litellm_core_utils/prompt_templates/factory.py:5097-5118` — unrelated-looking but motivated change: `add_cache_point_tool_block(tool, model=None)` and `_bedrock_tools_pt(tools, model=None)` now thread the `model` through so a Claude-4.5-on-Bedrock check can decide whether to attach a `ttl: "5m"|"1h"` field on the cache point. Wires through `is_claude_4_5_on_bedrock(model)`.
- `litellm/llms/bedrock/chat/converse_transformation.py`, `litellm/llms/bedrock/messages/invoke_transformations/anthropic_claude3_transformation.py`, `litellm/main.py` — call-site updates threading `model` into `_bedrock_tools_pt`.
- Three matching test files in `tests/test_litellm/...` cover the new helper, the cache-point TTL behavior, and the streaming-chunk-builder skip path.

## Analysis

The headline fix (the title) is the right kind of "found a real GIL hold via async-event-loop blocking" patch. The pre-fix `stream_chunk_builder` flow:
1. Receive streaming chunks from a provider (e.g., Claude extended thinking, OpenAI o1/o3, Gemini thinking).
2. The provider already supplies `usage.completion_tokens_details.reasoning_tokens` in the final chunk.
3. `ChunkProcessor.count_reasoning_tokens` runs anyway, calling `tiktoken.encode()` on every reasoning text chunk.
4. `tiktoken.encode()` is a C extension that holds the GIL — for 100k+ token reasoning responses, that's tens of seconds of event-loop block.
5. Downstream in `calculate_usage`, the recomputed value is **discarded** (the provider-supplied value wins) — so the entire recount was already pure-overhead-with-no-effect.

The new `chunks_have_reasoning_tokens` is the right gating predicate: it reports whether any usage chunk in the stream already carries `reasoning_tokens != None`, which is exactly the condition under which `calculate_usage` will discard the recount. Skipping `count_reasoning_tokens` in that case is provably semantics-preserving, not just an estimation tradeoff.

The defensive shape-handling in `chunks_have_reasoning_tokens` is over-engineered but justified: the codebase has *both* dict-shaped and `ModelResponse`-shaped chunks flowing through this builder depending on entry point (raw provider chunks vs. already-normalized chunks vs. chunks where `usage` was hoisted to `_hidden_params` for early routing decisions). The triple-fallback (`usage` direct → `getattr(chunk, 'usage')` → `chunk._hidden_params.get('usage')`) covers the three known shapes. A type union or single-shape normalization upstream would be cleaner long-term, but that's a refactor an order of magnitude larger than this fix.

The **bundled change is the concern**: the `add_cache_point_tool_block`/`_bedrock_tools_pt` model-parameter threading and the `is_claude_4_5_on_bedrock` TTL feature are completely orthogonal to the streaming-chunk reasoning-token fix. They touch four files (`factory.py`, `converse_transformation.py`, `anthropic_claude3_transformation.py`, `main.py`) and add a real new feature (per-model cache-point TTL). The PR description (which references "Re-opening previously closed PR #26245" against the new OSS staging branch) doesn't explicitly call out that this is two-features-in-one-trench-coat.

## Nits to address before merge

1. **Split the bundled cache-point-TTL change** into a sibling PR or, at minimum, a separate commit with its own subject line. As of the head SHA the commit history isn't visible from this review surface, but the reviewer-on-call shouldn't have to read 380 lines to find the 56 that match the title. If the staging branch process forces single-PR submissions, at least add a bolded **"Bundled change"** section to the description that names the cache-point-TTL work, links the issue it's fixing (if any), and quotes the matrix of `(model, cache_control.ttl)` cells the new code now honors.
2. **The triple-fallback chunk-shape sniffing in `chunks_have_reasoning_tokens`** deserves a comment naming the three known shapes and the entry points that produce each. Without that, a future contributor will see the third branch (`_hidden_params['usage']`) and "clean it up." Add `# Three shapes seen in practice: dict from raw SSE, ModelResponse from normalized stream, dict-with-usage-hoisted-to-_hidden_params from the early-routing path. Do not collapse without checking all three.`
3. **Symmetric `parallel_tool_calls=True` test missing**: the test scaffolding likely covers `reasoning_tokens` present, but a minor case to lock would be a chunk where `completion_tokens_details` is present but `reasoning_tokens` is `None` (provider returned the details object but didn't fill the field). The current early-return `if reasoning_tokens is not None: return True` correctly continues iterating in that case; a pin test would lock it.
4. **`is_claude_4_5_on_bedrock(model)` is a string-pattern match** (presumably) — confirm it handles both `bedrock/anthropic.claude-4-5-sonnet-...` and `bedrock-converse/anthropic.claude-4-5-...` route prefixes, since both code paths call `_bedrock_tools_pt` via different transformation modules.

## What I learned

The "GIL-hold blocks the event loop" failure mode in mixed sync-C-extension/async-Python codebases is one of the worst things to debug because the symptom (proxy liveness probe failures) is so far from the cause (tiktoken running on a 100k-token thinking response inside a coroutine that the runtime can't preempt). The fix here is also a nice case study in "the pre-existing code was strictly redundant work, not even an estimation/correctness tradeoff" — `calculate_usage` was already discarding the recount, so there was nothing to lose by skipping it. That kind of pure-removable-overhead is rare and worth shipping with the smallest possible diff; the bundled cache-point-TTL feature is the only thing diluting that win.
