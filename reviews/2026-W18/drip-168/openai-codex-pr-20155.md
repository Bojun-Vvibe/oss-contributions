# openai/codex PR #20155 — TUI: Remove core protocol dependency [2/6]

- Repo: openai/codex
- PR: https://github.com/openai/codex/pull/20155
- Head SHA: `c6d618d82be3d461adcac566d7c6a19fecc6b053`
- Author: etraut-openai
- Size: +107 / −379, 5 files
- Series: 2 of 6

## Context

This is the second slice of a six-PR series migrating the TUI off direct
dependence on `codex_app_server_protocol` event types in favor of consuming
notifications through `handle_server_notification`. The framing is "the TUI
should consume the protocol via notifications, not synthesize protocol
events from a parallel test-only event enum". The series target is to let
the protocol crate evolve without forcing TUI churn.

The specific surgery in this PR retires the `EventMsg::McpStartupUpdate`
and `EventMsg::McpStartupComplete` test-event variants, which were a
TUI-internal mirror of `McpServerStatusUpdatedNotification` used only to
drive snapshot tests. Production already routes MCP startup status through
`handle_server_notification(McpServerStatusUpdated, ...)`, so these
synthetic test events were paying duplicate maintenance cost.

## What the diff actually does

1. `codex-rs/tui/src/chatwidget.rs:7129-7132` deletes two arms from the
   master `EventMsg` match:
   `EventMsg::McpStartupUpdate(ev) => self.on_mcp_startup_update(ev)` and
   `EventMsg::McpStartupComplete(ev) => self.on_mcp_startup_complete(ev)`.
2. `codex-rs/tui/src/chatwidget/mcp_startup.rs` strips the
   `#[derive(Serialize, Deserialize)]` from `McpStartupStatus` (no longer
   needed once the test-event variant is gone), removes the test-only
   `serde` imports, and deletes the two `#[cfg(test)] fn on_mcp_startup_*`
   helpers that the deleted match arms called. The production
   `on_mcp_server_status_updated` handler stays.
3. `codex-rs/tui/src/chatwidget/test_events.rs` deletes
   `McpStartupUpdateEvent`, `McpStartupCompleteEvent`, and
   `McpStartupFailure` structs and the `EventMsg` variants that wrapped
   them.
4. `codex-rs/tui/src/chatwidget/tests.rs` deletes the
   `pub(super) use ... McpStartupStatus`, `McpStartupCompleteEvent`,
   `McpStartupUpdateEvent` re-exports.
5. `codex-rs/tui/src/chatwidget/tests/mcp_startup.rs` rewrites every test
   that used to fire `EventMsg::McpStartupUpdate{...}` /
   `EventMsg::McpStartupComplete{...}` to instead call new
   `notify_mcp_status` / `notify_mcp_status_error` helpers that build a
   real `ServerNotification::McpServerStatusUpdated(...)` and dispatch
   through `handle_server_notification`. This is the load-bearing test
   migration — every snapshot test now exercises the production code path.

## Risks and gaps

1. **The `complete_when_settled` flag in the deleted `on_mcp_startup_update`
   was passed `false` for the test path** but the production
   `on_mcp_server_status_updated` always passes `true` (or whatever
   "complete when settled" infers from the notification's content). This
   is the right cleanup — the test-only `false` was never reachable in
   production — but it does mean any test that previously relied on the
   "update without completing the round" semantics now goes through the
   production "complete-when-settled" path. The two tests retained
   (`mcp_startup_header_booting_snapshot`,
   `mcp_startup_complete_does_not_clear_running_task`) should be re-checked
   that their assertions still pin the same intent. From the diff
   excerpt visible the booting-snapshot test still fires only `Starting`
   (not `Ready`/`Failed`) so it shouldn't trip the settled path.

2. **`McpStartupFailure` is deleted entirely** — the new `notify_mcp_status_error`
   helper bundles the error string into the notification's `error: Some(_)`
   field. That's fine, but a follow-up test for the failure-status snapshot
   (which historically used the failed-server branch in
   `EventMsg::McpStartupComplete`) needs to confirm its assertions still
   capture the "what does the TUI render when an MCP server fails to
   start" surface. The diff shows the helper exists but I can't see all
   the rewritten tests — worth confirming snapshot diffs don't drift on
   the failed-server path.

3. **The `Serialize, Deserialize` removal from `McpStartupStatus`** is
   safe because there's no production caller serializing that enum — but
   if any downstream tool (a debug introspection panel, a TUI test
   harness consumed by another crate) was deserializing JSON snapshots
   into `McpStartupStatus`, this is a silent break. A `git grep
   McpStartupStatus` across `codex-rs/` would confirm scope.

4. **Series coordination risk**: this is [2/6] and the diff explicitly
   refers to `app-server-backed startup round settles` in the doc-comment
   change at `chatwidget.rs:880-887`. If [3/6] or later changes the
   `handle_server_notification` dispatch path or the
   `McpServerStatusUpdatedNotification` shape, the rewritten tests in this
   PR could go red. Worth landing the series as a stack and CI-gating each
   slice on the next so a partial revert doesn't leave the test suite
   untestable.

5. **The new `notify_mcp_status_error` helper is local-to-tests — it's not
   exported through `test_events.rs`**. That's fine, but if [3/6]+ wants
   to fire MCP failures from a different test file, the helper has to
   move. A `pub(super) use` from `tests/mcp_startup.rs` upward, or
   relocating the helper to `tests.rs` proper, would prevent duplicate
   helper definitions later in the series.

## Suggestions

- Confirm the failure-path snapshot tests (the ones that used to
  consume `EventMsg::McpStartupComplete { failed: vec![...] }`) still
  capture the rendered failure surface unchanged.
- `git grep -nP 'McpStartupStatus|McpStartupUpdateEvent|McpStartupCompleteEvent'`
  across the workspace to confirm no out-of-tree consumer was relying on
  the now-deleted types.
- Land the six PRs as a stack with each gating the next in CI so a
  partial revert doesn't leave the suite in a non-buildable state.
- Hoist `notify_mcp_status` / `notify_mcp_status_error` helpers into a
  shared test module (e.g. `tests/mod.rs` or `test_events.rs` proper)
  before [3/6] needs them.

## Verdict

**merge-after-nits** — clean dead-code removal that pulls the test path
onto the same notification surface production already uses, which is the
right direction. The nits are about series discipline and confirming no
downstream consumer relied on the deleted serializable enum, not about
the surgery itself.
