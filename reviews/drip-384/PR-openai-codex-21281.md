# openai/codex#21281 — Remove core MCP list tools op

- **Head SHA**: `9b9bbdc8b9502aa797f361e257075233fcd25315`
- **Stats**: +47 / -220, multi-file dead-code removal

## Summary

Subtractive cleanup that removes the `Op::ListMcpTools` submission and its handler chain (`list_mcp_tools` in `core/src/session/handlers.rs`, `collect_mcp_snapshot_from_manager` and `collect_mcp_snapshot_from_manager_with_detail` in `codex-mcp/src/mcp/mod.rs`) plus the `EventMsg::McpListToolsResponse` realtime-text mapping. Active MCP-tool-listing surface has migrated to the app-server JSON-RPC `mcp/list` endpoint per the broader app-server refactor; the core-side `Op::ListMcpTools` was the legacy in-process variant kept around for the test-suite's "submit a barrier op and wait for its response" pattern in `pending_input.rs`.

## Specific citations

- `codex-rs/codex-mcp/src/lib.rs:22`: removes `pub use mcp::collect_mcp_snapshot_from_manager;` re-export.
- `codex-rs/codex-mcp/src/mcp/mod.rs:34, 337-346, 573-608`: deletes the `McpListToolsResponseEvent` import and both the public `collect_mcp_snapshot_from_manager` wrapper plus the underlying `collect_mcp_snapshot_from_manager_with_detail` (the join of `list_all_tools` + `list_all_resources` + `list_all_resource_templates`).
- `codex-rs/core/src/session/handlers.rs:24-28, 517-545, 890-893`: deletes the `pub async fn list_mcp_tools(sess, config, sub_id)` handler and its single dispatch site `Op::ListMcpTools => list_mcp_tools(...)` in the submission loop. Note: the previous handler had `#[expect(clippy::await_holding_invalid_type, reason = "MCP tool listing reads through the session-owned manager guard")]` — that lint suppression goes away with the handler, which is correctness-positive (one fewer place where an `RwLockReadGuard` is held across an `.await`).
- `codex-rs/core/src/session/turn.rs:1508`: drops `EventMsg::McpListToolsResponse(_)` from the `realtime_text_for_event` no-text match arm. Correct because that variant can no longer be emitted from core.
- `codex-rs/core/tests/suite/pending_input.rs:166-174`: rewrites the test's "barrier" op from `Op::ListMcpTools` → `Op::GetHistoryEntryRequest{offset:0, log_id:0}` and the matching `wait_for_event` from `McpListToolsResponse` → `GetHistoryEntryResponse`. The `GetHistoryEntryRequest` op is a sensible barrier replacement — it's idempotent, low-cost, and round-trips through the same submission queue.

## Verdict

**merge-after-nits**

## Rationale

Clean subtractive change with the right barrier-test migration. Three nits worth a maintainer pass before merge: (1) `Op::ListMcpTools` and `EventMsg::McpListToolsResponse` are removed from the core handler but the variants themselves likely still exist in `codex-protocol` — if so, any out-of-tree TS/Python SDK that constructs `{"type":"ListMcpTools"}` will now hit a "no handler" / silent-drop path rather than a clean deserialization error; a `rg "ListMcpTools" codex-rs/` to confirm zero remaining references in the protocol crate (or a follow-up PR removing the variants there too) would close this. (2) `Op::GetHistoryEntryRequest{log_id: 0}` as a barrier — if a future refactor makes `log_id == 0` a sentinel for "no history" the test silently no-ops; a comment at `:166` naming "this is a barrier op, the values don't matter, pick any cheap op" would prevent future drift. (3) The `#[expect(clippy::await_holding_invalid_type, ...)]` removal is a quiet correctness win — worth a one-line callout in the PR body so reviewers know this PR isn't *just* dead-code deletion. No functional risk.
