# Review — openai/codex#20677

- PR: https://github.com/openai/codex/pull/20677
- Title: [codex] Emit MCP tool calls as turn items
- Head SHA: `b69ebeae9a8687cf4fb2f7445f4e49a77d1f9569`
- Size: +431 / −299 across 8 files
- Verdict: **needs-discussion**

## Summary

Refactors how MCP tool calls surface in the v2 app-server protocol:
- `app-server-protocol/src/protocol/event_mapping.rs` deletes the entire
  `EventMsg::McpToolCallBegin` and `EventMsg::McpToolCallEnd` arms
  (and their associated unit tests, ~241 deletions) that previously
  produced `ItemStarted`/`ItemCompleted` notifications with
  `ThreadItem::McpToolCall` payloads.
- New `protocol/v2.rs` (+114) and `protocol/items.rs` (+82) plumbing
  for emitting MCP calls through the generic turn-item path.
- `core/src/mcp_tool_call.rs` (+101 / −31) and
  `core/src/tools/handlers/mcp_resource.rs` (+49 / −25) shift to
  emitting items directly.

## Evidence / specific spots

- `app-server-protocol/src/protocol/event_mapping.rs` — `use` lines
  for `McpToolCallError`, `McpToolCallResult`, `McpToolCallStatus`
  removed; the two giant `EventMsg::Mcp*` match arms (~70 lines each)
  deleted; matching test block (`mcp_tool_call_begin_maps_to_*`,
  `mcp_tool_call_end_*`) removed.
- `protocol/protocol.rs` (+79) and `protocol/items.rs` (+82) — new
  item types presumably replace the deleted ones.

## Concerns (why needs-discussion)

1. **Silent capability change for downstream clients.** Any v2
   subscriber that pattern-matches on
   `ItemStarted/ItemCompleted` with `ThreadItem::McpToolCall` will
   stop seeing those events after this lands. The PR description does
   not appear to enumerate which clients are expected to migrate, nor
   whether the new `items.rs` types are wire-compatible. This deserves
   a CHANGELOG entry and/or a deprecation window before deletion.
2. **Lost test coverage.** Deleting ~241 lines of mapping plus the
   `mcp_tool_call_begin_maps_to_item_started_notification_with_args`
   and `_without_args` tests removes the only assertions that argument
   round-trip / null-args / `mcp_app_resource_uri` propagation work
   correctly. The replacement path in `core/src/mcp_tool_call.rs`
   needs equivalent tests covering: in-progress → completed transition,
   error-path mapping, `arguments: None` → `Value::Null`, and that
   `mcp_app_resource_uri` survives the new round-trip.
3. **`bespoke_event_handling.rs` only +4/−2** — a 2-line touch on a
   file with "bespoke" in the name suggests there might still be a
   parallel emission path. Reviewer should confirm there is no double
   emission (old path + new turn-item path) for some interim period.

## Recommended path forward

- Land the new turn-item emission first behind a feature flag or as
  additive output, deprecate the old `EventMsg::Mcp*` arms with a
  warning, then delete in a follow-up release. Or, at minimum, document
  in the PR description which app-server protocol version this lands in
  and confirm no in-tree consumer breaks.
- Re-add equivalent unit tests at the new layer before merging.
