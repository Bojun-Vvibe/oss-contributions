# BerriAI/litellm #26995 — fix(vertex): resolve Gemini tool results by call id

- **Repo:** BerriAI/litellm
- **PR:** #26995
- **URL:** https://github.com/BerriAI/litellm/pull/26995
- **Head SHA:** `a9d23cf0f52e3d7c3c8b7606c9993048bf331096`
- **Files touched:**
  - `litellm/llms/vertex_ai/gemini/transformation.py` (+22 -3)
  - `tests/litellm/llms/vertex_ai/gemini/test_transformation.py` (+59 -2)
- **Verdict:** merge-as-is

## Summary

Real correctness fix. Previously
`_gemini_convert_messages_with_history` resolved each `tool` role
message against `last_message_with_tool_calls` — i.e. whichever
assistant message *most recently* contained tool calls. That is wrong
when the conversation contains a sequence like:

```
assistant (tool_calls=[call_read_file])
assistant (text only, tool_calls=[])
tool      (tool_call_id=call_read_file)
```

The intervening text-only assistant message reset
`last_message_with_tool_calls` to itself (because the old guard
`assistant_msg.get("tool_calls", []) is not None` was always true —
even an empty list is not None), so the tool result was matched to the
wrong assistant message and the function-call linkage broke.

## Specific notes

- **`transformation.py:529-534`** — the old condition
  `assistant_msg.get("tool_calls", []) is not None or function_call is not None`
  evaluated `[] is not None == True`. The new form `tool_calls or
  function_call is not None` correctly treats an empty list as
  "no tool calls". This alone fixes the regression even without the
  ID map.
- **Lines 547-551** — populating `message_by_tool_call_id` only on the
  branch that already enters `convert_to_gemini_tool_call_invoke`
  ensures the map is populated only for assistant messages that
  actually emitted tool calls. No false positives.
- **Lines 600-613** — the lookup gracefully falls back to
  `last_message_with_tool_calls` when `tool_call_id` is absent or
  unmatched. Backward compatible for callers that don't supply tool
  call IDs.
- **Test at `test_transformation.py:96-152`** — exercises exactly the
  three-message scenario described above and asserts the resulting
  Gemini contents have the function_response in the correct slot.
  Excellent coverage.

## Why merge-as-is

The bug is real and reproduces in standard tool-using flows (any agent
that emits a "thinking" text message between tool call and tool
result). The fix is minimal, the test is precise, and the fallback
preserves prior behavior for clients without tool_call_id. Ship.
