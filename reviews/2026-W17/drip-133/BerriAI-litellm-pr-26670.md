# BerriAI/litellm #26670 — fix(otel): populate `gen_ai.output.messages` and `gen_ai.system_instructions` for Responses API

**Verdict:** `merge-after-nits`

- Repo: `BerriAI/litellm`
- PR: https://github.com/BerriAI/litellm/pull/26670
- Head SHA: `4d2e13c9`
- Closes: #25840
- Diff: +557 / -10 across `litellm/integrations/opentelemetry.py` (+92) and `tests/test_litellm/integrations/test_opentelemetry.py` (+465)

## What it does

Closes a real observability gap: `/v1/responses` (Responses API) calls were
emitting OTel spans missing `gen_ai.output.messages`,
`gen_ai.system_instructions`, and `gen_ai.response.finish_reasons`. Token
counts, costs, and *input* messages were being recorded; output content,
system prompt, and finish reasons were silently dropped because the
attribute-population code in `set_attributes()` only knew about the
`choices`-shaped chat-completions response.

## Design analysis

Three independent fixes folded into one PR, each at the right layer.

### 1. `gen_ai.system_instructions` coalescing (`opentelemetry.py:1681-1709`)

The previous code only checked `kwargs.get("system_instructions")` (Vertex
Gemini), missing two equivalent kwargs from sibling call paths:

```python
system_instructions = (
    kwargs.get("system_instructions")
    or kwargs.get("instructions")
    or kwargs.get("system")
)
```

The `or`-chain coalesces three names that carry the same semantic data
(`system_instructions` for Vertex AI Gemini, `instructions` for OpenAI
Responses, `system` for Anthropic Messages). The split string-vs-list
branch is necessary because the Responses API passes plain strings while
chat-completions and Anthropic pass message lists — the existing
`_transform_messages_to_otel_semantic_conventions` would mis-handle a raw
string.

Subtle correctness point worth flagging: this changes the implicit
precedence. If a request somehow has *both* `instructions` and `system`
(unusual but possible for a multi-provider router), the PR picks
`instructions` (because the `or` short-circuits in declaration order). Old
code would have picked neither, so the new behavior is strictly more
informative and not worse, but the precedence is now implicit in the
literal source order — worth a code comment.

### 2. `gen_ai.output.messages` for Responses API (`opentelemetry.py:1768-1791`)

Adds an `elif response_obj.get("output"):` branch after the existing
`choices` check, calling a new
`_transform_responses_api_output_to_otel(output)`. The new method at
`:1889-1934` walks the `output` list, handling both `type="message"` items
(emitting `{"role", "parts": [{"type": "text", "content": ...}]}`) and
`type="function_call"` items (emitting `{"type": "tool_call", "name",
"arguments", "id"}`). The output format matches the
`_transform_choices_to_otel_semantic_conventions` shape exactly — same
`{role, parts}` envelope, same part-type discriminator — which is the
right invariant: downstream consumers reading
`gen_ai.output.messages` shouldn't need to special-case Responses API
spans.

The `if not isinstance(item, dict): continue` and `if not isinstance(content,
dict): continue` defensive checks are appropriate for a
streaming/partial-response world where malformed items can land in
`output` mid-flight.

The `if text:` and `if parts:` skip-empty guards correctly avoid emitting
spurious empty-message records, which is the right semantic — an empty
output is the absence of an output, not an output that says "".

### 3. `gen_ai.response.finish_reasons` from `status` (`opentelemetry.py:1786-1791`)

Pulls `response_obj.get("status")` (e.g. `"completed"`) and wraps it in
the same `safe_dumps([status])` list shape used by the chat-completions
finish-reason path. List-of-one is the right shape — the OTel attribute
is conventionally a list because chat-completions can return multiple
choices, and Responses API returns one logical response.

### Test coverage (`test_opentelemetry.py:+459`)

The `TestOpenTelemetryResponsesAPI` class adds 21 unit tests covering the
right cells:

- `test_output_messages_populated_for_responses_api` (positive base case)
- `test_output_messages_with_multiple_content_items` (multi-part text)
- `test_output_messages_with_function_call` / `_mixed_message_and_function_call`
  (tool-call shape parity)
- Empty-text rejection
- The `_get_attr` helper at `:2952` is a clean way to extract the value
  the production code wrote without coupling to `set_attribute` call
  ordering — that's the right test infrastructure investment for a code
  path with ~30 attribute writes per call.

## Risks

1. **Status normalization gap.** The PR emits the raw Responses API status
   (`"completed"`, `"in_progress"`, `"failed"`, `"incomplete"`) directly as
   a finish reason. OTel `gen_ai.response.finish_reasons` semantic
   conventions expect normalized values like `"stop"`, `"length"`,
   `"content_filter"`, `"tool_calls"`. Mixing raw provider statuses with
   normalized chat-completion reasons in the same attribute will make
   trace dashboards harder to aggregate. Worth a normalization map (e.g.
   `"completed"` → `"stop"`, `"incomplete"` → `"length"` if the truncation
   reason is "max_tokens").
2. **`function_call` `arguments` may be a streaming-partial JSON string.**
   The transformer emits whatever string is in `item.get("arguments", "")`
   without validating it parses. For complete responses this is fine; for
   any partial-flush span, downstream consumers that try to parse the
   field will fail. Acceptable trade-off (don't validate, just record) but
   should be documented in the new method's docstring.
3. **No `reasoning` output-item type handled.** Responses API can include
   `type="reasoning"` items (when reasoning models are used). The PR
   silently drops them — they don't match `"message"` or
   `"function_call"`. May be intentional (reasoning content is sometimes
   sensitive) but should be an explicit policy comment, not an implicit
   else-branch fall-through.
4. **PR contains only Responses API tests.** No regression assertion that
   the existing `choices` path still works post-change. The
   `if response_obj.get("choices")` / `elif response_obj.get("output")`
   ordering is correct, but a one-line "chat-completions path still emits
   the same attributes" smoke test would lock in the no-regression promise.

## Suggestions

- Add a small `_RESPONSES_API_STATUS_TO_OTEL_FINISH_REASON` map for at
  least `completed → stop` and `incomplete → length` to keep dashboard
  aggregation sane across both APIs.
- Document the `function_call.arguments` "we record raw JSON string,
  caller validates" contract in `_transform_responses_api_output_to_otel`.
- Add an explicit `# reasoning items intentionally not emitted` comment if
  that's the policy, or a third `elif item.get("type") == "reasoning"`
  branch if it's an oversight.
- Add one chat-completions regression test alongside the new Responses API
  tests so future refactors of the `choices`/`output` branching can't
  silently break the older path.

## What I learned

This is the "we shipped a second response shape and the observability
layer didn't notice" pattern, which is recurring across LLM-router
codebases as `/responses` and tool-calling-first APIs land. The right
remediation isn't to add another `if isinstance(...)` switch — it's to
factor each shape's transform into a sibling method
(`_transform_responses_api_output_to_otel` next to
`_transform_choices_to_otel_semantic_conventions`) that emits the *same
output envelope*, keeping downstream consumers shape-agnostic. The PR
does exactly this. The remaining work — status normalization, parity
tests for the older path, and explicit policy on reasoning items — is
the lifecycle work that turns "fix the bug" into "establish the contract."
