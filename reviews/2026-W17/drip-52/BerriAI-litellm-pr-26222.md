---
pr: 26222
repo: BerriAI/litellm
sha: f503c061a59ed329de1967914e4e55124d352b8b
verdict: merge-after-nits
date: 2026-04-26
---

# BerriAI/litellm#26222 — fix(anthropic): json response_format + user tools non-streaming

- **URL**: https://github.com/BerriAI/litellm/pull/26222
- **Author**: Sameerlite
- **Files**:
  `litellm/llms/anthropic/chat/transformation.py`,
  `tests/test_litellm/llms/anthropic/chat/test_anthropic_chat_transformation.py`
  (+91/-20)

## Summary

When a non-streaming Anthropic call uses both
`response_format={"type": "json_schema", ...}` (which litellm
implements as a synthetic internal tool named
`RESPONSE_FORMAT_TOOL_NAME`) AND the caller's own user tools, the
old `_transform_response_for_json_mode` only triggered when
`len(tool_calls) == 1`. Result: if the model returned both the
internal json tool call AND a user tool call, neither path
worked correctly — the json payload wasn't extracted into
`message.content` and the internal tool call leaked back to the
caller as if it were a real tool. This PR splits the response
handling into a new `_resolve_json_mode_non_streaming` that:

1. If *all* tool calls are the internal json tool → behave as
   before (lift json into content, drop the tool calls).
2. If *some* tool calls are the internal json tool, others are
   user tools → strip the internal one, append its arguments to
   `text_content`, return only the user tool calls.
3. Otherwise → no change.

## Reviewable points

- `litellm/llms/anthropic/chat/transformation.py:1552-1591` —
  the new `_resolve_json_mode_non_streaming` returns a 3-tuple:
  `(replacement_message, filtered_tool_calls, extra_content_str)`.
  The shape is awkward but covers all three cases without
  branching at the call site. The all-internal branch
  (line 1573) returns `(_message, [], None)`; the mixed branch
  (line 1582-1585) returns `(None, filtered_tools, extra_content)`.
  Distinct return shape per case is fine because the sole caller
  (line 1968) destructures and uses each piece independently.

- `transformation.py:1577-1578` — guards the all-internal case
  on `arguments is None` and bails to `(None, tool_calls, None)`.
  This preserves the existing fall-through behavior for
  malformed internal calls (missing arguments). Same guard the
  old code had on line 1559-1560.

- `transformation.py:1959-1972` — caller integration. The merge
  is:

  ```python
  merged_text = text_content or ""
  if json_extra_content:
      merged_text = (
          merged_text + json_extra_content if merged_text else json_extra_content
      )
  ```

  Reasonable. Note: there's no separator between `text_content`
  and the appended JSON. If the model emits a chatty preamble
  before the structured output, the result is `"Here you
  go:{...json...}"` with no newline. **Nit**: insert `"\n"` (or
  `"\n\n"`) when both pieces are non-empty so callers can split
  trivially. Not a correctness blocker — `merged_text` is what
  the caller asked the model to return, but a tiny readability
  win.

- `transformation.py:1985` — when the all-internal branch fires,
  `_message = json_mode_message` replaces the constructed
  message and `stop_reason` is rewritten to `"stop"`. The mixed
  branch deliberately does *not* rewrite `stop_reason` because
  the user tool calls still need `stop_reason="tool_use"` so the
  caller knows to dispatch them. That distinction is correct and
  matches the test on line 153 of the test file (which asserts
  `filtered[0]["function"]["name"] == "get_weather"` and doesn't
  touch stop_reason).

- The unit test
  `test_anthropic_json_mode_non_streaming_mixed_internal_and_user_tools`
  (test file line 41-83) is a clean direct test of
  `_resolve_json_mode_non_streaming` with one internal + one user
  tool. Asserts `replacement is None`, `len(filtered) == 1`,
  `filtered[0]["function"]["name"] == "get_weather"`, and
  `extra == '{"answer": 42}'`. Tight.

## Risks

- The "extract json arguments string and treat it as content" is
  a bit of a stretch — JSON-as-content means the caller has to
  re-parse it. Existing all-internal path already does this
  via `_convert_tool_response_to_message`, so behavior is
  consistent with the legacy path. Fine.
- The mixed test only covers ONE internal + ONE user tool. A
  future test should cover the multi-user-tool variant
  (1 internal + N user) to lock in `filtered` ordering — the
  list comprehension on line 1586 preserves source order, which
  is the contract callers will assume.

## Verdict

`merge-after-nits`. Add the separator nit and ideally one more
test for multi-user-tool case; logic itself is correct and the
existing test covers the regression. The 3-tuple return is mildly
ugly but localized.

## What I learned

When a provider transformation injects a "synthetic" internal
tool to implement a higher-level concept (like
`response_format=json_schema`), the response-side cleanup must
handle the *mixed* case where the model invokes both the
synthetic tool and the caller's real tools in the same response.
A length-1 guard is a code smell that often hides this exact
class of bug.
