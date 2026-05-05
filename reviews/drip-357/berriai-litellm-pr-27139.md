# BerriAI/litellm #27139 — fix(vertex_ai/agent_engine): don't terminate stream on inner-action STOP (#19121)

- **Repo**: BerriAI/litellm
- **PR**: [#27139](https://github.com/BerriAI/litellm/pull/27139)
- **Author**: mateo-berri
- **Head SHA**: `f1e7ee2bc17d59a23f97b6a79b77dc09bd1b9d57`
- **Base branch**: `litellm_internal_staging`
- **Created**: 2026-05-04T22:30:49Z
- **Fixes**: #19121

## Verdict

`merge-after-nits`

## Summary of change

Fixes a real correctness bug in `litellm/llms/vertex_ai/agent_engine/sse_iterator.py`:
Vertex Agent Engine emits one SSE event per ADK action, and each action's
event carries `finish_reason: "STOP"` because STOP is the Gemini turn
terminator — not the Agent Engine stream terminator. The previous parser
mapped every `STOP` to OpenAI `finish_reason: "stop"`, which closed the
downstream stream wrapper after the very first inner action (typically a
`transfer_to_agent` function call), so the actual analyst response never
made it back to the caller.

The fix:

- `sse_iterator.py:36-69` — new `_extract_parts_from_chunk` static method
  walks `content.parts`, returning `(first_text_part, [tool_call_chunks])`.
  Handles both `functionCall` (REST/camelCase) and `function_call`
  (Python SDK/snake_case). Tool-call ids fall back to `f"call_{uuid.uuid4()}"`
  when ADK doesn't supply one.
- `sse_iterator.py:90-101` — finish-reason logic is rewritten:
  - if the chunk has tool_calls → `finish_reason = "tool_calls"`,
  - else if the chunk has text → preserve the prior `STOP → "stop"` mapping,
  - else → drop `finish_reason` entirely (intermediate / thought-only chunks
    must not terminate the stream).
- `sse_iterator.py:115-122` — delta is now built with explicit `delta_kwargs`
  so that text-only chunks set `content`+`role`, tool_call-only chunks set
  `tool_calls`+`role`, and content-less chunks emit `content=None` (a true
  no-op delta). Previous code emitted `content=text` (which was `None`) and
  `role="assistant" if text else None`, conflating "no text" with "no role".

Tests in `tests/litellm/llms/vertex_ai/agent_engine/test_transformation.py`:

- `:127-156` `test_chunk_parser_intermediate_function_call_does_not_finish_stream` —
  the regression test for #19121: a `transfer_to_agent` function_call chunk
  with `finish_reason: "STOP"` must surface as `finish_reason="tool_calls"`,
  not `"stop"`.
- `:160-176` `test_chunk_parser_intermediate_chunk_no_content_drops_finish_reason` —
  thought-only chunks (parts contain only a `thought_signature`) drop
  `finish_reason` entirely.
- `:179-208` `test_chunk_parser_camelcase_function_call` — REST API's
  `functionCall` casing is parsed equivalently to SDK's `function_call`.
- Existing two text-content tests are factored to reuse a `_iterator()` helper
  on `:77-81` (no behavior change).

## What's good

- Root cause is correctly diagnosed and the docstring on `:74-87` explains it
  unambiguously: "STOP is the Gemini-level terminator for that single action
  — it does NOT mean the Agent Engine stream is finished. The SSE stream
  ending is the only true end-of-response signal." Future maintainers will
  not relitigate this.
- The dual-casing handling (`function_call` vs `functionCall`) is the right
  defensive move; the test on `:179` pins it.
- The "thought-only chunk" branch (no text, no tool_calls) returning
  `finish_reason=None` is the precise fix for the reported bug — without it,
  the alternative (always-`finish_reason="tool_calls"` when STOP is seen)
  would mis-terminate intermediate non-tool chunks.
- Tool-call id fallback uses `uuid.uuid4()`, which is fine — but worth a
  follow-up: ADK's `id` field, when present, should be preserved as-is so
  consumers can correlate function_call → function_response, and the existing
  test on `:204` confirms that.
- Test reorganization (extracting `_iterator()`) is a tiny cleanup that
  meaningfully reduces duplication across 5 tests.

## Nits / questions

- **Multi-text-part chunks silently lose all but the first text part.**
  `_extract_parts_from_chunk` keeps only `text` from the first text-bearing
  part it sees (`:51-53`). If Vertex ever emits multiple text parts in a
  single action (rare today, but the schema permits it), the rest are
  dropped. Either concatenate text parts or add a TODO with a link to the
  spec. The previous code had the same bug, so this isn't a regression — but
  it's worth pinning down explicitly.
- **`finish_reason` mapping is asymmetric for non-STOP terminators with
  tool_calls present.** When `raw_finish_reason` is e.g. `"MAX_TOKENS"` and
  the chunk also has tool_calls, the new code sets
  `finish_reason="tool_calls"` and silently discards `MAX_TOKENS`. That may
  be the right OpenAI mapping (OpenAI's tool_calls finish_reason takes
  precedence in similar cases), but worth a one-line comment explaining the
  precedence choice on `:96-101`.
- Tool-call `index=len(tool_calls)` (`:64`) is correct *within* one chunk
  but doesn't account for tool-calls accumulated across chunks in a single
  stream. If two consecutive ADK actions each emit one tool_call chunk,
  both will arrive with `index=0` from this parser. If downstream stream
  reconstruction relies on monotonic global indexing, this will collide.
  Worth verifying against `BaseModelResponseIterator` semantics.
- The docstring talks about "downstream stream wrapper" closing — naming the
  exact wrapper (e.g. `CustomStreamWrapper`) and the place it inspects
  `finish_reason` would help the next reader.
- `usage_metadata = chunk.get("usage_metadata") or {}` (`:113`) — the `or {}`
  collapses a `None` value to `{}`. Previous code defaulted to `{}` directly.
  Functionally equivalent here; ignore.

## Sanitization

Test fixtures use `"id": "adk-redacted"` and `"id": "adk-1"`, no real ids or
secrets. The thought_signature value is `"..redacted.."`. Safe.

## Risk

Medium-low. The fix is targeted and well-tested for the reported failure
mode (#19121). Risk is mostly around edge chunks (multiple text parts,
non-STOP terminators with tool_calls); none of those are likely to be common
in production today.

## Recommendation

Land after (a) a one-line comment on `:96-101` clarifying the
`tool_calls`-takes-precedence mapping for non-STOP terminators and
(b) confirmation of cross-chunk tool_call `index` semantics with the
downstream stream wrapper.
