# BerriAI/litellm PR #26995 — fix(vertex): resolve Gemini tool results by call id

- URL: https://github.com/BerriAI/litellm/pull/26995
- Head SHA: `a9d23cf0f52e3d7c3c8b7606c9993048bf331096`
- Author: Genmin
- Verdict: **merge-after-nits**

## Summary

Fixes a long-standing correctness bug in the OpenAI→Gemini message conversion where a `tool` message would be paired with the *most recent* assistant message that issued tool calls, rather than the assistant message whose `tool_calls[].id` actually matches the `tool_call_id` on the result. This breaks down whenever a text-only assistant message intervenes between the tool call and the tool result — Gemini ends up referencing the wrong call site (or a missing one), and the resulting `function_response` carries the wrong `name`. Fix maintains a `Dict[tool_call_id → assistant_msg]` map across the conversion loop and looks up by id, falling back to the previous behavior only when the id is missing.

## Line-level observations

- `litellm/llms/vertex_ai/gemini/transformation.py` line 310: introduces `message_by_tool_call_id: Dict[str, ChatCompletionAssistantMessage] = {}`. Type annotation is precise. Good.
- `transformation.py` lines 530–533: pulls `tool_calls` and `function_call` out into locals. The truthiness change from `assistant_msg.get("tool_calls", []) is not None` to `tool_calls or function_call is not None` is **a subtle behavioral change**: the old condition was *always true* for `tool_calls` because `[]` is not `None`. The new condition treats an empty list as "no tool calls." This is almost certainly the right fix — an empty list shouldn't trigger the tool-call-invoke path — but it's a separate behavior change worth calling out explicitly in the PR description so reviewers don't miss it.
- `transformation.py` lines 547–551: the map is populated after `convert_to_gemini_tool_call_invoke` succeeds, and only for entries whose `id` is truthy. Correct ordering — we only register IDs we'll actually be able to look up. Note: `last_message_with_tool_calls = assistant_msg` is preserved as the fallback path, so older messages without `tool_call_id` continue to work.
- `transformation.py` lines 600–613: lookup logic is `message_by_tool_call_id.get(tool_call_id, last_message_with_tool_calls)` when `tool_call_id` is a non-empty string, else the legacy fallback. The `isinstance(tool_call_id_value, str)` guard is defensive — good given OpenAI typing isn't enforced at runtime here.
- `tests/litellm/llms/vertex_ai/gemini/test_transformation.py` lines 96–151: the new test reproduces the exact bug — an assistant tool-call message followed by a *text-only* assistant message, then a tool result. Asserts the `function_response.name` resolves to `read_file` (the original tool's name). Strong, targeted regression test.
- The test removes two unused imports (`openai`, `completion`); unrelated cleanup but harmless.

## Suggestions

1. Call out in the PR body that the truthiness check on `tool_calls` was tightened — reviewers skimming the diff will read it as cosmetic when it's actually behavioral.
2. Add a second test case where two assistant messages issue overlapping/sequential tool calls and a tool result targets the *earlier* one — this exercises the map's value over the fallback.
3. Consider also setting `message_by_tool_call_id` from `function_call` (single-call path) for the `function_call.name` style — currently only `tool_calls[]` populate the map, so a mixed history could still mis-resolve. Probably out of scope, but worth a `TODO`.
