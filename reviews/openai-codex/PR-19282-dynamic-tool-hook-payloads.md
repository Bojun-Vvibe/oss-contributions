# PR #19282 — dynamic tool hook payloads (PreToolUse / PostToolUse)

**Repo:** openai/codex • **Author:** pash-openai • **Status:**
DRAFT • **Net:** +123 / −0 (draft, originated from external
contributor Vincent K)

## Change

`DynamicToolHandler` (the codex tool plumbing for namespaced dynamic
tools like `openclaw__message`) previously didn't emit
`PreToolUsePayload` / `PostToolUsePayload` at all, so lifecycle
hooks couldn't observe dynamic tool calls. This PR adds both
`pre_tool_use_payload` and `post_tool_use_payload` impls on
`DynamicToolHandler`. The pre payload carries `tool_name`
(stringified `ToolName`) and `tool_input` (parsed JSON args). The
post payload adds `tool_use_id` and `tool_response` (a JSON object
with `contentItems` and `success`). Two unit tests cover the happy
path.

## Load-bearing risk

Three concerns, all centered on the contract this exposes to user
hooks:

1. **`tool_name` collapses the namespace separator.** The pre/post
   payloads pass `invocation.tool_name.display()` to
   `HookToolName::new(...)`. The test confirms the result is
   `"openclaw__message"` — a flat string with `__` between
   namespace and name. This is fine if hooks treat it as opaque,
   but any hook that wants to dispatch on namespace will end up
   string-splitting on `__`, which (a) is fragile if a future
   namespace allows underscores and (b) prevents hooks from
   filtering by namespace cheaply. A structured
   `{ namespace, name }` would be more future-proof.

2. **`tool_response` is a synthetic JSON envelope, not the real
   model-visible output.** It's `{ "contentItems": result.body,
   "success": result.success_for_logging() }`. That `success` is
   *for logging* per its own name — not necessarily aligned with
   the model's view of "did this tool call succeed." Hooks that
   gate side effects on `tool_response.success` will be reading
   logging-flavored truth, which can drift from model-flavored
   truth.

3. **`tool_input` is the parsed JSON, but if `parse_arguments`
   fails, both `pre_tool_use_payload` and `post_tool_use_payload`
   silently return `None` (via `?`).** That means malformed-args
   tool calls *skip* hook notification entirely — including hooks
   designed precisely to *observe* malformed calls (auditing,
   anomaly detection). The hook contract should probably surface
   `tool_input: serde_json::Value::Null` or a raw-string variant
   rather than silently dropping the event.

The draft state and the "external contributor + OpenAI maintainer
co-author" framing suggests this is mid-review; these are the
points the reviewer should be raising before non-draft.

## Concrete next step

(1) Switch `tool_name` to a structured `{ namespace, name }` (or
add it alongside the flat string for migration). (2) Use the
model-visible success signal, not `success_for_logging`, and
document which one this is. (3) Emit pre/post events even on
arg-parse failure with an explicit `tool_input: null` + an `error`
field so hooks can observe failed-pre-validation calls. (4) Add a
test for the arg-parse-failure path that asserts the hook **does**
fire (with whatever degraded payload the design picks).

## Verdict

Right idea, hooks need this. Draft is the right state — at least
three of the contract decisions deserve explicit calls before
freezing them.

## What I learned

Hook payload schemas are surprisingly load-bearing: every
`Option<Payload>` returning `None` is an invisible "this event
disappears." For observability/audit hooks, a degraded payload is
almost always better than a missing event. The `success_for_logging`
naming smell is also a useful tell — when the type system
distinguishes "for logging" from "actually true," any new consumer
needs to pick deliberately.
