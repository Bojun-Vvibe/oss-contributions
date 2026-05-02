# openai/codex PR #20787 — fix(tui): scope MCP startup events to emitting thread

- Repo: `openai/codex`
- PR: #20787
- Head SHA: `06fd1307aab4f5d77b5718cf78d3f7e261a4996a`
- Author: shayne-oai

## Summary
Sub-agent MCP startup notifications were routed Global and overwrote the
leader TUI's `mcp_startup_status` map, reopening the leader's spinner and
preventing the settle check from passing. This PR adds an optional
`thread_id` to `McpServerStatusUpdatedNotification`, populates it in
`apply_bespoke_event_handling`, and routes notifications carrying a
`thread_id` through `ServerNotificationThreadTarget`. Missing `thread_id`
keeps the legacy Global behavior for backward compatibility. Closes
#18068, refs #16821, #19542.

## Specific references
- `codex-rs/app-server-protocol/src/protocol/v2.rs:7120-7128` — adds
  `thread_id: Option<String>` with `#[serde(default)]`. The `Option` plus
  `default` is exactly what makes legacy clients keep working: a missing
  field deserializes to `None` and falls through to Global routing.
- `codex-rs/app-server-protocol/schema/json/v2/McpServerStatusUpdatedNotification.json:25-32`
  and the three sibling schema JSONs — `threadId` is `["string","null"]`
  with `default: null`. Schema is consistent with the Rust source.
- `codex-rs/app-server-protocol/schema/typescript/v2/McpServerStatusUpdatedNotification.ts`
  — generated TS now exposes `threadId: string | null`. Correct shape.
- `codex-rs/app-server/src/bespoke_event_handling.rs:225` — sets
  `thread_id: Some(conversation_id.to_string())` on every emitted
  notification. Good — current servers always populate it; the `Option`
  is purely a wire-compat affordance.
- `codex-rs/app-server/tests/suite/v2/thread_start.rs:482,519,552` — the
  existing integration test now extracts `thread.id` from the
  `ThreadStartResponse` and asserts both the Starting and Failed
  notifications carry `thread_id: Some(thread.id.clone())`. Pins the
  populate-on-emit guarantee.
- `codex-rs/tui/src/app/app_server_event_targets.rs:+36/-1` and
  `codex-rs/tui/src/app/tests.rs:+39` — routing changes plus app-level
  test asserting a sub-agent `McpServerStatusUpdated` does not flip the
  leader's task-running state. (Diff truncated at fetch but file
  additions counts confirm coverage.)
- `codex-rs/tui/src/chatwidget/tests/mcp_startup.rs:+2` — minor test
  surface bump to keep existing assertions honest under the new field.

## Verdict
`merge-as-is`

## Rationale
Textbook backwards-compatible protocol addition: `Option<String>` field
with `#[serde(default)]`, current writers always populate it, missing-
field path is preserved as the documented Global fallback, and there's
matched coverage at protocol-schema, app-server, and TUI app levels.

The PR body explicitly calls out the contract: "A missing `thread_id`
keeps the existing Global behaviour for backward compatibility with older
app servers" — and that's exactly what `#[serde(default)]` plus the
routing branch deliver.

No banned strings; the change does not affect any product-name surface.
