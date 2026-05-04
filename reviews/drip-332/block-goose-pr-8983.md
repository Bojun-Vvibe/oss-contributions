# block/goose PR #8983 — fix: notify clients of generated session names

- **Link:** https://github.com/block/goose/pull/8983
- **Head SHA:** `6cab656232064992915444579d1b5f4b77999863`
- **Files:** `crates/goose/src/acp/server.rs`, `crates/goose/src/acp/server/sessions.rs`, plus session-naming pipeline glue
- **Diff size:** +~250 / −smaller (676 diff lines including test/fixture noise)

## Summary
Pipes generated/renamed session names through the ACP `SessionInfoUpdate` notification stream so connected clients see the title change without polling. Introduces a per-connection mpsc channel populated when the agent generates a session name and consumed by a tokio task that fans out `SessionUpdate::SessionInfoUpdate { title, updated_at, meta }` notifications.

## Specific citations
- `crates/goose/src/acp/server.rs:43-52` — import block extended to include `SessionInfoUpdate` (was previously absent). Mechanical and necessary.
- `crates/goose/src/acp/server.rs:271-298` — new `spawn_session_name_update_notifier(cx: ConnectionTo<Client>) -> UnboundedSender<SessionNameUpdate>`:
  - Builds an `unbounded_channel`. **Risk:** `unbounded_channel` will grow without bound if the notification consumer (the tokio task) lags behind. For session-name updates this is unlikely to overwhelm the system, but it is worth noting — a `bounded(64)` with `try_send` + warn on `Full` would fail-loud rather than fail-OOM.
  - Spawns a tokio task that loops over `rx.recv()` and forwards each update as a `SessionNotification`.
  - On send failure logs a `warn!` with `thread_id` and `error` — good observability.
- `crates/goose/src/acp/server.rs:1186-1205` — wires `session_name_update_tx` into the `Agent::with_config` builder via `.with_session_name_update_tx(session_name_update_tx)`. Guarded by `(!disable_session_naming).then(...)` so the channel is only created when naming is enabled. Good — no overhead in the disabled case.
- `crates/goose/src/acp/server/sessions.rs:143-160` — the `rename_session` handler now also calls `self.session_manager.update(&internal_session_id)` after the thread title change so internal session metadata stays in sync. Previously the thread title and the session metadata could drift.

## Observations
- The notification carries `title`, `updated_at`, and `meta` — that matches what a client typically needs to refresh a sidebar entry. Confirm the ACP schema version on consumers can decode `SessionInfoUpdate` (this is gated by the imported `SessionInfoUpdate::new()` constructor; older clients without that variant will reject the message).
- The `cx.send_notification(...)` call at `server.rs:289` returns a `Result`. The current code logs a warn and continues. If the underlying connection is dead, the loop will keep receiving updates and dropping them. That is acceptable, but add a check to break out of the loop when the connection is closed (e.g. on a specific `error` variant) so the task does not linger for the lifetime of the process.
- The mpsc receiver is moved into the spawned task; sender is held by the agent. When the agent is dropped, the channel closes and the loop exits cleanly. Lifetime story is sound.

## Nits
- Replace `unbounded_channel` with a `channel(64)` (or similar) and warn on `try_send` failures. Session-name generation is rare enough that 64 is generous.
- Add a `tracing::debug!` at `server.rs:280` when a name update is enqueued — useful when debugging "why didn't my client see the name update".
- The `spawn_session_name_update_notifier` task has no name. Tag it with `tokio::task::Builder::new().name("acp-session-name-notifier")` (where supported) for easier flame-graph attribution.

## Verdict
`merge-after-nits`
