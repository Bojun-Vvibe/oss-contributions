# openai/codex #19509 — Record MCP tool result telemetry on spans

- **Repo**: openai/codex
- **PR**: [#19509](https://github.com/openai/codex/pull/19509)
- **Head SHA**: `6e660eefa8b4c2c32d178cd234567930fd1efb77`
- **Author**: mchen-oai
- **State**: DRAFT (+211 / -23)
- **Verdict**: `merge-after-nits`

## Context

`mcp.tools.call` spans currently capture only request-side
attributes (server, tool, call ID, connector, session, turn). MCP
servers know things the client can't: the resolved target identity
behind a tool, whether the call kicked off a user-facing flow,
etc. This PR introduces a server→client telemetry side-channel via
`CallToolResult.meta["codex/telemetry"]["span"]` that the client
allowlists and records on the active span.

## Design

The control structure change in `handle_approved_mcp_tool_call`
(`codex-rs/core/src/mcp_tool_call.rs:317-356`) is the most
interesting bit: previously the future was instrumented inline with
`.instrument(mcp_tool_call_span(...))`; now the span is built first,
held by name, the future is instrumented with `span.clone()`, and
after the future resolves the code calls
`record_mcp_result_telemetry(&span, result.as_ref().ok())`. This
matters because once `instrument(...)` consumes the span you can't
later `record(...)` on it, so the explicit clone-and-hold is the
correct pattern.

Allowlist + bounds at lines 487-528:

- The lookup chain `meta → "codex/telemetry" → "span" → object`
  fails closed at every step (`Some(...) else { return; }`), so a
  malformed or absent meta block is a no-op rather than a panic
  source.
- `target_id` is `as_str()` only, filtered against empty string,
  truncated to `MCP_RESULT_TELEMETRY_TARGET_ID_MAX_CHARS` (256) via
  `truncate_str_to_char_boundary` so a 1MB attacker-controlled
  string from a hostile MCP server can't blow up span memory or
  trip downstream attribute-size limits.
- `did_trigger_user_flow` is `as_bool()` only — strings like
  `"true"` are silently dropped, which is the right contract.
- The span fields are pre-declared as `Empty` in
  `mcp_tool_call_span` (lines 444-460), so `span.record(...)` after
  the fact actually attaches them. This is a tracing-rs gotcha that
  the PR gets right.

## Risks

1. **Schema discoverability.** The contract is "MCP server may
   return `_meta["codex/telemetry"]["span"]` with `target_id` and
   `did_trigger_user_flow`" but it lives in a Rust constant block
   with no public docs. An MCP server author has no signal that
   this hook exists. Worth a short note in the MCP integration
   docs (or at minimum a doc comment on
   `record_mcp_result_telemetry` explaining the wire shape).

2. **`as_object()` on `result.meta`.** If a server returns
   `_meta` as a non-object JSON value (number, array), it's
   silently dropped — correct, but worth a single-line comment
   given how easy it is to misread the chain.

3. **No metric for "telemetry was recorded".** When debugging why
   a span is missing target info, you can't tell from the client
   side whether (a) the server didn't send it, (b) the meta key
   was wrong, or (c) the type check failed. A `tracing::debug!`
   on the dropped-due-to-type-mismatch branch would help on-call
   without changing public behavior.

## Suggestions

- Add a `tracing::debug!` on the type-mismatch fall-throughs (e.g.
  when `target_id` exists but `as_str()` returns `None`).
- Doc-comment `record_mcp_result_telemetry` with the wire-format
  example and the truncation/type contract.
- Consider whether `target_id` should also have a "low-cardinality"
  contract note — span attributes with high-cardinality values
  cost a lot at the backend, and `target_id` sounds like the kind
  of thing a server might naively populate with a UUID-per-call.

## What I learned

The instrument-then-record pattern (build span, clone into
`instrument`, record on the original after the future completes)
is the canonical way to attach late-binding attributes in
tracing-rs. Most "I can't `record` on my span" bugs in async code
trace back to letting `instrument` consume the only handle.
