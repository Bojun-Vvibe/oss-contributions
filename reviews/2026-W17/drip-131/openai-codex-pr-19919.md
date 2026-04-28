# Review — openai/codex #19919

- **Title**: app-server: notify clients of remote-control environment changes
- **Author**: euroelessar (Ruslan Nigmatullin)
- **Head**: `33fafff106ae616b747ba832e842c6ebaefc17a2`
- **Verdict**: merge-as-is

## Summary

Adds a v2 server notification `remoteControl/environment/updated` carrying `environmentId: string | null` so app-server clients learn when the remote-control environment id transitions (account change, stale enrollment invalidation, successful re-enrollment) — explicitly *not* on transient websocket disconnects. Distribution uses a Tokio `watch` channel on `RemoteControlHandle` rather than adding a new transport-event variant; the current non-null id is replayed to newly initialized clients.

## Findings

- **`watch` channel is the right primitive** — `tokio::sync::watch` only delivers the *latest* value, so a client that initializes during a flurry of enrollment churn doesn't replay every intermediate state, just the current one. Correct for an identity claim where stale-but-future-valid intermediates would just confuse downstream cache logic.
- **Replay-on-initialize at `transport/remote_control.rs`** (sending current non-null id after initialize) handles the bootstrap race correctly — without this, clients that initialize *after* the transition broadcast would keep the cleared/old id forever. The `non-null` filter is also right: replaying a `null` post-init would cause clients to falsely believe enrollment was just cleared.
- **Single-arm `Option<String>` payload** is the right wire shape — distinguishes "not enrolled / cleared" (`null`) from "enrolled with id X" (`Some("...")`) without inventing a tagged enum that consumers would have to switch over. Easy to extend later by promoting to a struct.
- **Generated artifacts kept in sync**: `ServerNotification.json` (legacy + v2 schemas), `codex_app_server_protocol.schemas.json`, `v2.schemas.json`, `typescript/v2/RemoteControlEnvironmentUpdatedNotification.ts`, `typescript/v2/index.ts`, and the union in `ServerNotification.ts` all updated in one PR — no half-regeneration risk.
- **TUI no-op match arm** is the right defensive move — `ServerNotification` is exhaustive in core code so the TUI must handle the new variant or stop compiling. Explicit no-op signals "TUI deliberately ignores remote-control identity transitions" rather than future-developer confusion about a missing handler.
- **Test coverage cited** (`cargo test -p codex-app-server transport::remote_control --lib`, `env RUST_MIN_STACK=8388608 cargo test -p codex-app-server --lib`) covers the broadcast path and the post-init replay; without seeing the test bodies the verification commands are at least the right targets.
- **Field naming consistency**: `environmentId` (camelCase via `serde(rename_all = "camelCase")`) on the JSON wire matches the rest of the v2 protocol; the Rust field is `environment_id` so no surprise for either side. The protocol description text "Current remote-control environment id exposed to remote-control clients" is the right one-line summary.

## Recommendation

Merge as-is. Architecturally sound, generated artifacts are in sync, the `watch`-channel + replay-on-init shape correctly handles both broadcast-during-churn and bootstrap-race semantics, and the TUI no-op arm preserves exhaustive-match safety without committing to UI surfacing this notification yet.