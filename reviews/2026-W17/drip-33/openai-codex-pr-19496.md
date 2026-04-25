# Review: openai/codex#19496 — Streamline MCP handlers

- **PR**: https://github.com/openai/codex/pull/19496
- **State**: OPEN
- **Author**: pakrym-oai
- **Range**: +151 / −236
- **Head SHA**: `03c30b8825ea9c07c54240aec30c6695b1a6604a`
- **Base SHA**: `5308b025eeca89bb6970c4214507f8cb51c451ab` (stacked on top of #19495)
- **Verdict**: merge-as-is
- **Reviewer date**: 2026-04-25

## What the PR does

Refactors the MCP-side request handlers in
`codex-rs/app-server/src/codex_message_processor.rs` (model list,
experimental feature list, MCP server refresh/status/OAuth, MCP
resource read, MCP tool call). The pattern is consistent: extract
the body into an `async { ... }.await` block (or a sibling
`*_response()` helper) that returns
`Result<Response, JSONRPCErrorError>`, then a single
`outgoing.send_result(request_id, result).await` at the boundary.
Net change is −85 lines, but the win is in flattening — every
"on error: build `JSONRPCErrorError`, `send_error`, `return`"
ladder collapses into `?`.

## Observations

1. **The `model_list` rewrite is structurally correct.** In the
   diff at `codex_message_processor.rs:5042–5092`, the early
   `total == 0 → empty response` branch and the cursor parse
   (`invalid_request(format!("invalid cursor: {cursor}"))`) and
   the `start > total` overflow check all preserved. End cursor
   semantics (`end < total → Some(end.to_string())`) unchanged.
   The clamp `effective_limit.max(1).min(total)` is preserved
   verbatim — important because `limit=0` previously couldn't
   stall pagination, and it still can't.
2. **`experimental_feature_list_response` split is the right
   shape for testing.** The handler is now thin — just builds
   the result and forwards. The body lives in
   `experimental_feature_list_response(&self, params) -> Result<…>`
   which can be unit-tested without an `outgoing` channel mock.
   Worth landing future tests against this surface.
3. **`bespoke_event_handling.rs` aligns the helper imports.**
   The diff drops the raw `INTERNAL_ERROR_CODE` /
   `INVALID_REQUEST_ERROR_CODE` imports in favor of
   `internal_error(...)` / `invalid_request(...)` helpers
   (presumably from `error_code.rs`). This means new code can't
   accidentally construct a `JSONRPCErrorError` with a wrong
   code — there's only one path. Good downstream effect.
4. **Behavior on the `mcp_server_refresh` / OAuth paths is
   preserved.** I checked the `pending_rollback` send-error
   site (`bespoke_event_handling.rs:2316–2319`) — it still uses
   `send_error` directly because the request_id arrives async
   from a different state machine, not from the immediate
   handler. Author called this out in the PR body. Correct
   call.
5. **Risk: stack ordering.** This PR is stacked on #19495,
   which is stacked on #19493. Since they all touch
   `codex_message_processor.rs`, conflicts are guaranteed if
   one merges out of order. The author should land them in
   order, or rebase if any earlier slice changes.
6. **Nit (not blocking):** the `Ok::<_, JSONRPCErrorError>(...)`
   turbofish in `model_list` (line ~5083) is only needed
   because the surrounding `async` block can't infer the error
   type. Could be cleaner as a typed helper return, but the
   turbofish is fine and lets the change stay localized.

## Verdict reasoning

Pure mechanical refactor with verified test coverage cited in
the PR body (`v2::mcp_resource`, `v2::mcp_tool`,
`v2::mcp_server_status`). No semantic change, no error-class
drift, error codes preserved one-for-one, fail-loud behavior
unchanged. Land it.

## What I learned

The "extract `async { ... }.await` returning `Result<R, E>` and
funnel through `send_result`" pattern is a clean way to
incrementally lift JSON-RPC servers off the
`send_response`/`send_error` style without rewriting the
transport. Worth stealing for similar codebases that have
accumulated `if err { send_error; return }` ladders.
