# Review: BerriAI/litellm #27279 — Fix responses tool conversion for chat bridge

- Carrier: `BerriAI/litellm`
- PR: #27279
- Head SHA: `78b773f5`
- Drip: 387

## Verdict
`merge-after-nits` (man)

## Summary
Two distinct fixes in `litellm/responses/litellm_completion_transformation/
transformation.py:1384-1426` to the `transform_responses_api_tools_to_chat_completion_tools`
helper, which is the bridge that lets a Responses API tool definition be
used against a Chat Completions backend (Anthropic, Vertex, etc.).

1. **Read Chat-Completion-style nested `function: {...}` tool fields.**
   The previous code only read the flat `typed_tool.get("name")` /
   `typed_tool.get("description")` / `typed_tool.get("parameters")` /
   `typed_tool.get("strict")` shape. If the caller (e.g., a client SDK
   that pre-formats for Chat Completions) passed `{"type":"function",
   "function": {"name":..., "parameters":...}}`, every field came out
   as the empty default and the bridge silently produced
   `{"name":"", "description":"", "parameters":{"type":"object"},
   "strict":false}`. Fix introduces `function_dict = cast(Dict[str, Any],
   tool.get("function") or {})` and uses `function_dict.get(...) or
   typed_tool.get(...)` precedence at every field, with the explicit
   `cast(Dict[str, Any], ...)` to satisfy type-check.
2. **Drop unsupported Responses-only tool types (`custom`, `shell`)
   instead of forwarding them.** The previous catch-all `else` branch
   would pass these through verbatim to the Chat Completions endpoint,
   which then rejects the request with a 400. Fix adds an explicit
   `elif tool.get("type") in {"custom", "shell"}: continue` to silently
   drop them, with a parallel test asserting only the `function` tool
   survives a mixed-type input.

## Diff anchors
- `transformation.py:1387` — `function_dict` extraction. The `or {}`
  guard handles `tool.get("function") == None` correctly.
- `:1389-1393` — `parameters` precedence chain
  `function_dict.get("parameters") or typed_tool.get("parameters", {})
  or {}` — first non-empty wins, then the unconditional
  `parameters["type"] = "object"` injection (preserved from prior code,
  required by Anthropic's tool schema).
- `:1395-1399` — `strict` precedence specifically tests for `is not None`
  rather than truthy, so an explicit `strict: false` in the nested
  `function` dict overrides a top-level `strict: true`. Subtle but
  correct.
- `:1402-1410` — output dict construction uses the new precedence chains.
  The `or ""` safety net for `name` and `description` is preserved
  (so a fully-empty input still produces a valid-shaped output rather
  than a `KeyError`).
- `:1425` — new `elif tool.get("type") in {"custom", "shell"}: continue`.
  The set-literal lookup is O(1) and the `continue` short-circuits to
  the next iteration without falling into the catch-all `else`.
- Test additions at `test_litellm_completion_responses.py:1192-1239`:
  - `test_transform_function_tools_reads_chat_completion_style_function_fields`
    asserts `result_tool["function"] == function_tool["function"]` —
    full round-trip equality including `strict: True`.
  - `test_transform_unsupported_responses_api_tool_types_are_dropped`
    feeds `[custom, shell, function]` and asserts `len(result_tools)
    == 1` with only the function surviving. The flat-shape function
    in this test (`name` at top level, no `function` dict) verifies
    the legacy path still works.

## Concerns / nits
1. **The `{"custom", "shell"}` deny-list is open-ended and undocumented.**
   The Responses API has more tool types in flight (file_search,
   web_search, code_interpreter, computer_use, etc.) — some of which
   already round-trip via the existing catch-all. A code comment
   listing *which* Responses-API tool types are intentionally
   forwarded vs dropped would prevent the next maintainer from adding
   a new `mcp_tool` type and accidentally letting it 400 the request.
2. **No deprecation/warn path for dropped tools.** A user who passed
   a `custom` tool gets it silently removed and may wonder why their
   tool never fires. A `verbose_logger.warning("Dropping
   unsupported tool type %s for chat-completions bridge", tool.get(
   "type"))` would make this debuggable.
3. **`strict` precedence is `function_dict.get("strict") if
   function_dict.get("strict") is not None else typed_tool.get(
   "strict", False)` at `:1395-1399`.** This calls `.get("strict")`
   twice; a single-pass `_strict = function_dict.get("strict"); strict
   = _strict if _strict is not None else typed_tool.get("strict",
   False)` is marginally faster and clearer. Pure micro-readability.
4. **The flat-vs-nested precedence is `function_dict` first, then
   `typed_tool`.** This is the right call when both are present
   (the more-specific nested form wins) but worth a one-line code
   comment so future maintainers don't flip it.
5. **No test for the "both flat and nested set" conflict case.** A
   fixture like `{"type":"function", "name":"flat",
   "function":{"name":"nested"}}` would lock the precedence contract.

## Risk
Low. Both fixes are surgical and additive; the legacy flat-shape path
is preserved (verified by the second test, which uses the flat shape
for its `get_weather` tool). The `{"custom", "shell"}` drop list is
intentionally narrow and unlikely to cause regression for existing
callers.
