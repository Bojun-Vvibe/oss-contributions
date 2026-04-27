# PR #26632 — fix(streaming): emit assistant role on empty completions (immediate EOS)

- **Repo**: BerriAI/litellm
- **PR**: #26632
- **Head SHA**: `2eb9ffd1`
- **Author**: alighazi288
- **Size**: +85 / -1 across 2 files
- **URL**: https://github.com/BerriAI/litellm/pull/26632
- **Verdict**: **merge-as-is**

## Summary

Closes #26428: when an upstream provider (the bug repro names SGLang +
Qwen3-Coder-480B) sends an empty / immediate-EOS streaming completion
as exactly three SSE chunks — chunk 1 carries `role:"assistant"` with
empty content (filtered upstream by `is_chunk_non_empty`), chunk 2
carries `finish_reason:"stop"` with no role and no content, chunk 3
carries usage only — litellm's synthesized empty-delta finish chunk
ended up with `role: None`, so callers downstream (proxy, OpenAI SDK
clients) couldn't identify the message role and the response became
indistinguishable from a malformed delta. Fix carries role on the
finish chunk if and only if no earlier chunk has carried it.

## Specific changes

- `litellm/litellm_core_utils/streaming_handler.py:1058-1067`: in the
  `_is_delta_empty` branch of `return_processed_chunk_logic`, the
  previously hardcoded `Delta(content=None)` is replaced with a built
  `delta_kwargs` dict:
  ```python
  delta_kwargs: Dict[str, Any] = {"content": None}
  if self.sent_first_chunk is False:
      delta_kwargs["role"] = "assistant"
      self.sent_first_chunk = True
  model_response.choices[0].delta = Delta(**delta_kwargs)
  ```
  The `sent_first_chunk` toggle is the existing mechanism the
  streaming handler uses for "have we already emitted role" — flipping
  it here also prevents a later content chunk (in a more complex stream
  shape that did the same trick) from re-emitting role. Comment
  explicitly names "SGLang where the only role-bearing chunk had empty
  content and was filtered upstream" + cross-references issue #26428.

## Tests

- `tests/test_litellm/litellm_core_utils/test_streaming_handler.py:2095-2138`
  — `test_role_surfaced_on_finish_chunk_for_empty_completion_sglang`:
  builds a `ModelResponseStream` with `Delta(role=None, content=None)`
  and `finish_reason="stop"`, sets `sent_first_chunk=False` to
  simulate the upstream-filtered-role-bearing-chunk-1 scenario, calls
  `chunk_creator(chunk=finish_chunk)`, asserts the result has
  `delta.role == "assistant"`, `finish_reason == "stop"`, and
  `sent_first_chunk` flipped to `True`.
- `:2143-2168` — companion test
  `test_role_not_duplicated_on_finish_chunk_when_content_arrived`:
  same shape but with `sent_first_chunk=True` (already emitted), asserts
  `delta.role is None` on the finish chunk so role doesn't get
  re-emitted on the trailing chunk for normal streams. This is the
  critical regression-guard that pins the *non-emit* path and stops
  a future "always emit role on finish" naive fix from sneaking in.

## Risks

- **Test fixture name `initialized_custom_stream_wrapper`** is shared
  with sibling tests in the file — assumes the fixture provides a
  `CustomStreamWrapper` with `chunk_creator` callable. This is
  correct based on the surrounding tests in the file (the diff
  references several similarly-shaped tests above and below). If
  the fixture default `sent_first_chunk` is `True`, the first test
  must explicitly set it to `False` (which it does at `:2110`); if
  the default is `False`, the second test must explicitly set it to
  `True` (which it does at `:2153`). Both are explicit — pin holds.
- **Provider-specific scope**: the comment names SGLang but the fix
  is provider-agnostic — any upstream that filters the role-bearing
  empty content chunk will now get correct role surfacing. That's
  the right scope; gating on `custom_llm_provider == "sglang"` would
  be a footgun (every other provider with the same misshape would
  silently keep the bug).
- **Two-arm test coverage** is exactly the right pattern: one test
  pins "emit role when not yet emitted", the other pins "do not
  re-emit when already emitted". Without the second test, a future
  fix that says "always emit role on finish chunk" would pass the
  first test and silently double-emit role on every normal stream's
  finish chunk.

## Verdict

**merge-as-is**: 10-line behavior change, one toggle on existing
state, paired with the canonical two-arm test coverage. The bug
reproduces against a real provider in the wild, the fix is at the
right layer (post-`is_delta_empty`, where the synthesized empty
delta is already being constructed), the cross-reference to #26428
is in the code comment so future readers can find the bug context.

## What I learned

The "synthesize an empty delta" code paths in streaming handlers are
where role/content/finish_reason invariants get silently broken
because the synthetic chunk doesn't carry the upstream's role
attribution. The fix here — gate role emission on `sent_first_chunk`
boolean — relies on that boolean being maintained correctly across
*all* paths that emit non-empty deltas. Any code path that emits a
delta with role and forgets to flip `sent_first_chunk = True` would
cause this fix to *re-emit* role redundantly. The companion
`test_role_not_duplicated_on_finish_chunk_when_content_arrived`
test at `:2143-2168` is what makes this bug-shape harder to
re-introduce — it explicitly asserts the negative case, which is the
side that's invisible until a downstream parser breaks on duplicate
role assignments. Generally: when fixing a "missing X" bug by adding
a conditional emit, always pair the fix with a test on the
"don't-emit-twice" path, otherwise the second-order regression is
just waiting to happen.
