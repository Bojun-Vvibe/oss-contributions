# BerriAI/litellm #26995 — fix(vertex): resolve Gemini tool results by call id

- URL: https://github.com/BerriAI/litellm/pull/26995
- Head SHA: `8f025227a6eee857dd34c443b0e9be3d19d0a467`
- Files: `litellm/llms/vertex_ai/gemini/transformation.py` (+19/-3), `tests/litellm/llms/vertex_ai/gemini/test_transformation.py` (+59/-2)
- Closes #26965

## Context / problem

When the OpenAI-shape conversation history is converted to Gemini's `Content[]` shape in `_gemini_convert_messages_with_history`, tool results need to be reattached to the assistant message that *declared* the tool call (so the converter can recover the function name + arg-shape for the `function_response` part). The previous implementation tracked exactly one back-pointer:

```python
last_message_with_tool_calls = assistant_msg
```

If the model emitted assistant tool-call → assistant text-only → tool result, the text-only intermediate didn't carry tool calls but the original code's "is this an assistant turn at all" path overwrote `last_message_with_tool_calls` with `None` (or the converter simply lost the binding to the right tool-call origin). Either way the later tool result mapped to the wrong / missing assistant message and the request blew up before reaching Vertex AI. This is a real shape — the model frequently narrates ("I will inspect the file and then summarize it.") between calling a tool and the tool returning.

## Design analysis

Two surgical edits in `transformation.py`:

### 1) Build a tool-call-id → assistant-message index — `:307-310`, `:547-551`
```python
message_by_tool_call_id: Dict[str, ChatCompletionAssistantMessage] = {}
...
if tool_calls:
    for tool_call in tool_calls:
        tool_call_id = tool_call.get("id")
        if tool_call_id:
            message_by_tool_call_id[tool_call_id] = assistant_msg
last_message_with_tool_calls = assistant_msg
```
Index is populated only when an assistant message actually has tool calls. The `last_message_with_tool_calls` back-pointer is preserved (assignment kept at `:551`) — so the new index is *additive* and the existing fallback behavior (when a tool result has no `tool_call_id`, e.g. legacy / non-OpenAI-shape callers) is unchanged.

### 2) Resolve tool result by `tool_call_id` first, fallback second — `:600-610`
```python
tool_result_message = messages[msg_i]
tool_call_id = tool_result_message.get("tool_call_id")
message_with_tool_call = (
    message_by_tool_call_id.get(tool_call_id, last_message_with_tool_calls)
    if tool_call_id is not None
    else last_message_with_tool_calls
)
_part = convert_to_gemini_tool_call_result(tool_result_message, message_with_tool_call)
```
Lookup discipline is correct: only consult the index when `tool_call_id is not None` (don't pollute the lookup with `None` keys); when the id is present but doesn't match anything in the index (out-of-band tool result), fall back to `last_message_with_tool_calls` rather than crashing.

The other edit at `:528-534` is also a real bugfix bundled in:
```python
tool_calls = assistant_msg.get("tool_calls")
function_call = assistant_msg.get("function_call")
if (tool_calls or function_call is not None):
```
The previous condition `assistant_msg.get("tool_calls", []) is not None` was always-truthy when `tool_calls` defaulted to `[]` (an empty list is not `None`) — the assistant-tool-invoke arm fired even for assistants with zero tool calls. The new `tool_calls or function_call is not None` is the truth-discriminating shape.

### Test — `test_transformation.py:96-148`
The fixture is exactly the failure shape:
1. user "Read the file before answering."
2. assistant with `tool_calls: [{id: "call_read_file", function: {name: "read_file", arguments: {...}}}]` and `content: None`
3. **assistant with text content "I will inspect the file..." and `tool_calls: []`** (the intermediate that broke the old path)
4. tool result with `tool_call_id: "call_read_file"`

Asserted output is the correct three-Content shape: user → model (function_call + text) → user (function_response with `name: "read_file"` recovered from the index). If the lookup-by-id had failed and fallen back to `last_message_with_tool_calls`, the function_response wouldn't be able to recover the function name — so this test is dispositive of the index actually being consulted.

The unrelated `from litellm.types.llms import openai` and `from litellm.types import completion` imports removed at `:13-14` are unused-import cleanup — fine.

## Risks

- The index is per-conversation-conversion (local to one call), so memory is bounded by message count — no leak.
- Same `tool_call_id` reused across two different assistant messages would silently overwrite the first index entry. OpenAI's contract is "ids are unique within a conversation" so this should never happen, but a `if tool_call_id in message_by_tool_call_id: warn` would catch broken upstreams. Not blocking.
- Fallback to `last_message_with_tool_calls` when the id misses preserves the legacy bug shape for that one path — arguably it should raise instead, since a tool result with an id-not-in-history is itself a contract violation. But "preserve existing behavior on the unhappy path" is a defensible call for a focused fix.

## Suggestions

- Add a second test where the tool result has a `tool_call_id` that doesn't match anything in the conversation (covers the fallback arm).
- Add a test for two interleaved tool calls (`call_a`, `call_b`, text, tool_result for `call_a`, tool_result for `call_b`) — locks the "index resolves the *right* one, not just the most recent" invariant which is the whole point of the change.
- Consider extracting `message_by_tool_call_id` population into a small helper so the assistant-message branch isn't 50+ lines.

## Verdict

`merge-after-nits` — correct fix for a real Vertex/Gemini conversion bug introduced by the model emitting a text-only assistant turn between a tool call and its result, with the right additive shape (index + fallback, not index-replaces-fallback), the right truth-discriminator on the tool-call branch (`tool_calls or function_call is not None`), and a dispositive regression test for the failure shape from the linked issue. Wants two more test cases (id-miss fallback, interleaved calls) before merge.
