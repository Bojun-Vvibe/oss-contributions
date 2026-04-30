---
pr-url: https://github.com/BerriAI/litellm/pull/26917
sha: d82aa1e4940e
verdict: merge-as-is
---

# fix(vertex-ai): recover tool-call history when Responses API conversion drops tool_calls

Targeted fix at `litellm/llms/vertex_ai/gemini/transformation.py:308-316` in `_gemini_convert_messages_with_history`: a pre-pass walks `messages` once and builds `tool_call_map: dict[tool_call_id → assistant_message]` for every assistant message that has a non-empty `tool_calls` array. Then at the tool-response handling site `:605-608`, when iterating into a `tool` / `function` role message, the code now looks up `messages[msg_i].get("tool_call_id")` in the map and overrides `last_message_with_tool_calls` with the *originating* assistant message rather than relying on the stateful "last seen" variable. The companion guard at `:540-543` widens the assistant-tool-call detection from `assistant_msg.get("tool_calls", []) is not None` to `... is not None and len(...) > 0` so an assistant message with `tool_calls: []` no longer poisons the `last_message_with_tool_calls` cursor.

The bug shape is excellent: a third-party CLI emits an assistant message with `"tool_calls": []` (empty list, not absent) followed by a *separate* assistant message carrying the real `tool_calls`, then a `tool` response referencing the second message — the prior code's stateful cursor pointed at the empty-tool_calls message at response-time, so name recovery failed and Vertex returned 400 with the misleading "Missing corresponding tool call for tool response message". The 60-line regression test at `test_vertex_ai_gemini_transformation.py:1916-1973` reproduces the exact 4-message sequence and asserts the converted `function_response` part has `name == "shell"` (the correct originating tool name, not whatever fell out of the empty-list arm).

The `tool_call_map` fix is the right *shape* here — recovering the originating message by ID is a content-addressed lookup that's robust to message reordering, empty-list quirks, and any future provider that sends multi-step assistant turns. Doing it as a pre-pass is `O(n)` extra over the message list, which for any realistic conversation is sub-microsecond. Could have been done as a lazy `dict.setdefault` build inside the main loop, but the pre-pass is more readable and the perf is irrelevant.

## what I learned
"Last seen X" stateful cursors in a parser are a recurring antipattern: any input that violates the "X always precedes use-of-X" assumption breaks them silently. Replace with a content-addressed map (`id → record`) built from a pre-pass — same complexity, robust to reordering, and the bug surface shrinks to "missing ID" which is loud and easy to test.
