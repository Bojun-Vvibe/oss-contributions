# Review — openai/codex#20787

- PR: https://github.com/openai/codex/pull/20787
- Title: fix(tui): scope MCP startup events to emitting thread
- Head SHA: `5163a068721b920dea6e6b0a3c874437de10f552`
- Size: +127 / −3 across 11 files
- Verdict: **merge-after-nits**

## Summary

`McpServerStatusUpdatedNotification` previously had no thread context,
so when multiple threads in one app-server process ran their own
`McpConnectionManager`, every thread's TUI saw every other thread's
MCP startup spinners. This PR threads a `thread_id: Option<String>`
field through the v2 protocol schema, sets it on the bespoke event
emitter, and updates the TUI's notification router to dispatch
`McpServerStatusUpdated` to the matching thread (falling back to
global when the field is absent, for legacy clients).

## Evidence

- `codex-rs/app-server-protocol/src/protocol/v2.rs:7120-7128` adds
  `pub thread_id: Option<String>` with `#[serde(default)]` to keep
  legacy clients deserialising. The accompanying schema files
  (`codex_app_server_protocol.schemas.json:10970`,
  `codex_app_server_protocol.v2.schemas.json:7623`,
  `v2/McpServerStatusUpdatedNotification.json:26`,
  `v2/McpServerStatusUpdatedNotification.ts:7`) are regenerated
  consistently.
- `codex-rs/app-server/src/bespoke_event_handling.rs:226` populates
  `thread_id: Some(conversation_id.to_string())` at emission time —
  i.e., emitters always set it; only legacy clients deserialise
  `None`.
- `codex-rs/tui/src/app/app_server_event_targets.rs:143-146` moves
  `McpServerStatusUpdated` from the "always Global" arm into a
  thread-aware arm that returns `notification.thread_id.as_deref()`.
- Two new unit tests at the bottom of
  `app_server_event_targets.rs` (the
  `mcp_server_status_updated_without_thread_id_is_global` and
  `..._with_thread_id_routes_to_thread` cases) lock the legacy and
  modern routing in place.
- `codex-rs/app-server/tests/suite/v2/thread_start.rs:482-553`
  destructures `ThreadStartResponse { thread, .. }` and asserts
  `failed.thread_id.as_deref() == Some(thread.id.as_str())` for both
  the `Starting` and `Failed` notifications.

## Notes / nits

- The TS export at `v2/McpServerStatusUpdatedNotification.ts`
  reorders `thread_id` to the first field of the struct literal
  (because of how ts-rs emits docstring-prefixed fields). That's a
  cosmetic change to a generated file — fine — but downstream
  consumers that destructure positionally will be unaffected because
  TS only positional-destructures arrays. Worth a sanity check that
  the Stainless / typed-client emit doesn't pin field order in
  weird ways.
- `Some(conversation_id.to_string())` allocates per emission. Given
  MCP startup events are bounded by the MCP server count and only
  fire during connect/disconnect, the cost is negligible — but if
  this pattern spreads to high-frequency events later, an
  `Arc<str>` for `conversation_id` would amortize the alloc.
- The `thread_id` is `Option<String>` rather than `Option<ThreadId>`.
  That matches the rest of the protocol (which serialises thread IDs
  as strings on the wire) but means TUI code stringifies-then-
  compares. Not a blocker; consistent with sibling notifications.

Ship after the schema-emission spot-check.
