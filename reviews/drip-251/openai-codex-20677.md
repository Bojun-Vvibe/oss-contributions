# openai/codex#20677 — [codex] Emit MCP tool calls as turn items

- **PR:** https://github.com/openai/codex/pull/20677
- **Head SHA:** `d5db19fca07777502061461a20f7a3c9102495bd`
- **Stats:** +321 / -58 across 7 files (`protocol/items.rs`, `protocol/protocol.rs`, `core/src/mcp_tool_call.rs`, `core/src/tools/handlers/mcp_resource.rs`, `app-server-protocol/v2.rs`, `thread_history.rs`, `app-server/bespoke_event_handling.rs`)

## Context
`McpToolCall` was the last app-server item still synthesized from deprecated legacy `McpToolCallBegin` / `McpToolCallEnd` events. Recent migrations moved item ownership into core `TurnItem`s. This PR completes the pattern: MCP tool calls now follow the same canonical `ItemStarted` / `ItemCompleted` lifecycle, and the legacy events are kept as compatibility fanout for older clients only.

## Design
Five coordinated edits at the right boundaries:

1. **Core type** — new `TurnItem::McpToolCall(McpToolCallItem)` at `codex-rs/protocol/src/items.rs:+43` carrying optional completion fields (`result: Option<Result<CallToolResult, String>>`, `duration: Option<Duration>`, `mcp_app_resource_uri`). The `Option` wrapping on `result` is the load-bearing correctness point: it's the typed representation of "in-progress vs. completed-ok vs. completed-error" replacing the stringly-implicit "Begin event seen but End not yet".

2. **Status projection** — at `app-server-protocol/v2.rs:6493-6520` (the `From<CoreTurnItem>` impl), the three-arm match derives `McpToolCallStatus`:
   ```rust
   let status = match &mcp.result {
       None => McpToolCallStatus::InProgress,
       Some(Ok(result)) if !result.is_error.unwrap_or(false) => McpToolCallStatus::Completed,
       Some(Ok(_)) | Some(Err(_)) => McpToolCallStatus::Failed,
   };
   ```
   Note the `Some(Ok(result)) if !is_error` guard correctly classifies MCP-protocol-level errors (the upstream returned a result with `is_error: true`) as `Failed` rather than `Completed`. That's an easy bug to write — `Ok(result)` looks like success but `CallToolResult` carries its own error flag.

3. **Emitter swap** — `core/src/mcp_tool_call.rs:185-195` replaces the direct `EventMsg::McpToolCallBegin` send with `notify_mcp_tool_call_started(...)`, and the completion path at `:363-380` swaps `EventMsg::McpToolCallEnd` for `notify_mcp_tool_call_completed(...)`. Both helpers (presumably) emit the canonical `ItemStarted` / `ItemCompleted` for the new `TurnItem::McpToolCall` *and* fan out the legacy event for compat — which is what the PR description says.

4. **Legacy fanout suppression in app-server v2** — `app-server/src/bespoke_event_handling.rs:836-841` carves `EventMsg::McpToolCallBegin(_) | EventMsg::McpToolCallEnd(_)` out of the deprecated-event handler with the explicit comment that v2 receives the canonical `TurnItem::McpToolCall` lifecycle instead. This is the "no double-notify" guard — without it, v2 clients would see both the legacy events and the new item lifecycle for the same call. Critically, thread-history *replay* still uses the legacy event path (deliberate, called out in PR body), which is right because historical sessions persisted the legacy events not the new items.

5. **Resource-tool emitter parity** — `core/src/tools/handlers/mcp_resource.rs:+18/-25` updates the MCP resource tool emitter to match. This is the parity that prevents one of the two MCP-tool entry points from drifting.

## Tests
`v2.rs:10459-10523` adds two dispositive `From` conversion tests:
- An in-progress fixture (`result: None`, `duration: None`) → `ThreadItem::McpToolCall { status: McpToolCallStatus::InProgress, ..., duration_ms: None }`.
- A completed fixture (`result: Some(Ok(CallToolResult { is_error: Some(false), ... }))`, `duration: Some(Duration::from_millis(42))`) → `status: McpToolCallStatus::Completed, duration_ms: Some(42), result: Some(Box::new(...))`.

Both lock the projection contract end-to-end. **Missing:** a third fixture for `result: Some(Ok(CallToolResult { is_error: Some(true), ... }))` to lock the "MCP-level error → Failed not Completed" guard at `v2.rs:6498`. That's the most-likely-to-regress arm.

## Risks
- The `is_error: Some(true)` → `Failed` arm is the easiest to silently regress under refactor. **Add a third fixture**.
- `duration_ms: i64::try_from(duration.as_millis()).ok()` saturates to `None` on overflow rather than `i64::MAX`. For sub-month durations that's fine; if anyone runs a 292M-year tool call it returns `None` instead of clamp. Worth one-line comment that overflow is treated as missing-data not maxed-out.
- `arguments.unwrap_or(JsonValue::Null)` at `:6517` projects "no arguments" as `Null`. If clients distinguish "Null arguments" from "missing arguments key", this loses information. Probably fine for MCP semantics but worth a sentence.
- Thread-history replay still using the legacy event path is correct *for now*; if persistence ever moves to the new item shape, the dual-source becomes a fork that needs explicit reconciliation. Note in protocol docs.

## Suggestions
1. Third `From` test for `is_error: Some(true)` → `Failed`.
2. One-line comment at `:6493-6499` explaining why `Ok(result)` with `is_error: true` is `Failed` not `Completed`.
3. PR body could explicitly note the legacy-event channel is *only* fanned out (not silenced) so external clients reading the deprecated events keep working — currently inferred from the comment but not stated up front.

## Verdict
**merge-after-nits** — clean lifecycle promotion at the right boundaries, correct dual-emit-with-v2-suppression discipline, dispositive tests for the two main arms but missing the MCP-error arm.

## What I learned
"`Result<T, E>`-then-`is_error` flag inside `T`" is a sneaky double-discriminator pattern that always needs explicit branching. The MCP wire protocol uses an inner `is_error` boolean on a successful upstream response to indicate semantic error, distinct from a transport-level failure. The status projection has to recognize both. The other lesson: when migrating an event-stream API to a typed-item API, the right pattern is dual-emit during the transition (canonical item + legacy event fanout) with explicit suppression at any consumer that would double-process — this PR does both, including the carve-out at `bespoke_event_handling.rs:836` and the deliberate retention of the legacy path for thread-history replay.
