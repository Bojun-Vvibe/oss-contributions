# openai/codex#21277 — [mcp] Return Accept early per feedback

- **Head SHA**: `076cc009c11aefab89b1a7fe911cc9672f36bca1`
- **Author**: mzeng-openai (Matthew Zeng)
- **Stats**: +59 / -0, 2 files

## Summary

Short-circuits MCP server elicitation requests when the connection manager's `elicitations_auto_deny()` flag is set, returning a synthetic `ElicitationResponse { action: Accept, content: Some({}), meta: None }` immediately rather than queuing a UI prompt. Adds a focused tokio test verifying the auto-accept path and that no event is emitted to the session bus.

## Specific citations

- `core/src/session/mcp.rs:14-24`: the new early-return block reads `services.mcp_connection_manager.read().await.elicitations_auto_deny()` and returns `Some(ElicitationResponse { action: ElicitationAction::Accept, content: Some(serde_json::json!({})), meta: None })`.
- The PR title says "auto_deny" but the action returned is `Accept` — confirmed correct against the body comment ("Return Accept early when auto_deny is enabled per feedback") and the test name `request_mcp_server_elicitation_auto_accepts_when_auto_deny_is_enabled` at `tests.rs:299` — but this naming is genuinely confusing.
- Test at `tests.rs:298-340` calls `set_elicitations_auto_deny(true)` then `session.request_mcp_server_elicitation(..., McpServerElicitationRequest::Form { meta: None, message: ..., requested_schema })` and asserts `Some(ElicitationResponse { action: Accept, content: Some(json!({})), meta: None })` plus `rx.try_recv().is_err()` — pinning both the response shape and the no-event-emitted invariant.
- New imports at `tests.rs:77, 128` (`McpElicitationSchema`, `ElicitationAction`) properly scoped.

## Verdict

**needs-discussion**

## Rationale

The implementation is mechanically correct and the test pins the contract well. **However**, the semantic mismatch between flag name (`elicitations_auto_deny`) and returned action (`Accept`) is a real wart that warrants maintainer ack: when an admin sets a "deny everything" toggle, returning `Accept` with empty content `{}` could actually grant the server-requested capability silently if any consumer treats `Accept + empty content` as approval-with-defaults rather than rejection. The PR body's "per feedback" hint suggests an upstream review chain settled this, but without seeing the original feedback an external reviewer can't verify the contract. Recommend: either rename to `elicitations_auto_accept_empty` or document at the flag definition why deny semantics are encoded as `Accept{}`. Test coverage is also single-shape (Form only) — the `Confirmation`/`Choice` variants of `McpServerElicitationRequest` aren't exercised, though the early-return covers them all by construction.

