---
pr_number: 26670
repo: BerriAI/litellm
head_sha: 9928618788389e623aa0c57b9249d084bfe72482
verdict: merge-after-nits
date: 2026-04-28
---

# BerriAI/litellm#26670 — fix(otel): populate gen_ai.output.messages and gen_ai.system_instructions for Responses API

**What changed.** +760/−10 across 2 files. Three correlated semantic-conventions gaps closed in `litellm/integrations/opentelemetry.py` plus 21 new unit tests. Closes #25840.

1. **System instructions tri-source coalescing** at `:1681-1715`. The old code only checked `kwargs.get("system_instructions")` (Vertex Gemini path). New code coalesces three kwarg names that carry the same semantic data via nested `is not None` ternaries (correctly chosen over truthiness so an empty `[]` doesn't fall through):
   - `system_instructions` (Vertex Gemini chat-completion)
   - `instructions` (OpenAI Responses API)
   - `system` (Anthropic Messages API)
   
   Plus an `isinstance(system_instructions, str)` short-circuit at `:1696` that writes the raw string without running `_transform_messages_to_otel_semantic_conventions` (which expects message-list shape). Without this short-circuit, a plain-string Responses API `instructions` would be coerced through the message-transformer and emitted as garbled JSON.

2. **Output messages elif-arm** at `:1774-1809`: new `elif response_obj.get("output"):` branch after the existing `choices` check, dispatching through new helper `_transform_responses_api_output_to_otel(output_items)` at `:1922-?` that converts `[{type: "message", content: [{type: "output_text", text: ...}]}]` items into the same `{role, parts}` shape `_transform_choices_to_otel_semantic_conventions` produces. Reuses `OpenTelemetry._tool_calls_kv_pair` for `function_call` items via a small adapter shape `{"function": {"name": ..., "arguments": ...}}` mimicking `ChatCompletionMessageToolCall` so per-tool-call span attribute parity with the chat-completion path is preserved.

3. **Finish reasons fallback** at `:1822-1828`: emits `[response_obj.get("status")]` (e.g. `["completed"]`) for the Responses API since there are no `choices[].finish_reason` to iterate.

**Why it matters.** Any downstream OTel consumer (Datadog, Honeycomb, Phoenix, Fiddler) currently sees Responses API spans with token counts + cost attributes populated but `gen_ai.output.messages` / `gen_ai.system_instructions` / `gen_ai.response.finish_reasons` empty. Existing dashboards built on those attributes silently report "no output content" for the entire `/v1/responses` traffic class — a real observability bug, not a polish item.

**Concerns.**
1. **The chained `is not None` ternary at `:1689-1697`** is correct but reads as "first non-None wins" precedence (`system_instructions > instructions > system`). Document this explicitly in the comment block — a future maintainer adding a fourth source (e.g. Bedrock `system_prompt`) would want to know whether to slot it in front or behind the chain. Better still, extract to `_coalesce_system_instructions(kwargs)` with the precedence list as a module-level constant.
2. **`hasattr(item, "get")` rather than `isinstance(item, dict)`** at `:1949,1958` is a deliberate accommodation of `BaseLiteLLMOpenAIResponseObject` Pydantic models exposing `.get()`. Document this explicitly with a one-line comment naming why `isinstance(dict)` is wrong here — otherwise a reflex "tighten the type check" PR will silently break the Pydantic path.
3. **Defensive `if not hasattr(item, "get"): continue`** at `:1948,1956` swallows malformed items silently. For an observability layer this is correct (don't crash the request path), but a `verbose_logger.debug(...)` on the skip would be useful for diagnosing why output is silently truncated.
4. **`elif response_obj.get("output"):` is mutually exclusive with the `choices` branch** — confirm no LLM in the wild returns *both* `choices` and `output` (e.g. some compatibility-shim provider). If it does, this PR silently picks `choices` and drops `output`. Probably OK for the current provider matrix, worth a brief check.
5. **`status` → `finish_reasons` value mapping is direct,** so `status=in_progress` would be emitted as a finish reason — but this code path runs in `set_attributes()` which is called at completion time, so `status` should always be terminal (`completed`/`failed`/`cancelled`). Verify with a unit test pinning a `status=in_progress` request not emitting the attribute (or emitting it correctly).
6. **`function_call` adapter shape at `:1797-1803`** hard-codes `name=""` and `arguments=""` defaults. If `_tool_calls_kv_pair` treats empty strings as valid emitted attribute values, the OTel span will get spurious `gen_ai.tool.name=""` entries on malformed input. Skip-empty would be safer.
7. **21 new tests is excellent** — particularly the `TestTransformResponsesAPIOutput` cell for non-dict items / empty output / missing `call_id`. Verify the regression cell mentioned in the PR body ("existing `choices`-based responses still work") actually covers the previously-broken `system` kwarg path now newly populating attributes that downstream tests may assert on as previously-absent.

Right shape, real bug, well-tested. Ship after the precedence-doc nit (concern 1) and confirming no provider returns both `output` and `choices` (concern 4).
