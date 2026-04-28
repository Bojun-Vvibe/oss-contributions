# openai/codex#19919 — app-server: notify clients of remote-control environment changes

- Head SHA: `33fafff`
- Author: euroelessar
- Files: 16 / +363 / −27

## Summary

Adds a new v2 server notification `remoteControl/environment/updated` carrying `environmentId: string | null` and wires it through the app-server lifecycle — published when enrollments are loaded, created, cleared for account changes, or invalidated by the backend. Exposes the lifecycle from `RemoteControlHandle` via a Tokio `watch` channel (chosen over a transport event so late subscribers automatically see the latest non-null id). Newly-initialized app-server clients receive a replay of the current non-null id immediately after `initialize` so they can reconcile local state with the remote-control backend contract.

## Specific observations

- Protocol surface added at `codex-rs/app-server-protocol/schema/json/v2/RemoteControlEnvironmentUpdatedNotification.json` (new file, 14 lines) with a single `environmentId: string | null` property. Symmetrically threaded into `ServerNotification.json:5354-5373`, `codex_app_server_protocol.schemas.json:4348+/12523+`, the v2 schemas file at `codex_app_server_protocol.v2.schemas.json:9177+/10927+`, and the generated TypeScript at `schema/typescript/ServerNotification.ts:38` (import) + the union type literal at `:70` (the load-bearing wide-union diff). Six-surface symmetry across schema/v2-schema/v2-file/ts-import/ts-union/everything-else is correct — partial commits across schema generators have been a recurring source of drift in this repo.
- The TUI carve-out is the right defensive pattern — `EventMsg::RemoteControlEnvironmentUpdated(_) => {}` (or equivalent exhaustive-match no-op, inferred from PR body since the diff was truncated) on the TUI side keeps `cargo check -p codex-tui` green so the new variant doesn't break the unrelated TUI compilation while the TUI itself doesn't yet care about the value.
- `Tokio watch` channel choice over a transport event is the load-bearing design call. Rationale (from PR body): late-subscribing clients automatically get the latest `Some(id)` value via `watch::Receiver::borrow()` without needing a separate "fetch current" RPC. The "send current non-null id to a client after initialize" hook is the replay path that operationalizes this — without it, a client that connects between two id transitions would see no id until the next change.
- `environmentId: string | null` (nullable) is the correct shape — the absence-of-enrollment cell needs to be distinguishable from "never reported", and `null` does that within a non-optional field. A future `unset` event doesn't need a separate notification variant.
- The diff truncates at the protocol/schema layer in my review — I haven't directly verified the `published when enrollments are loaded, created, cleared for account changes, or invalidated by the backend` callsites in `transport::remote_control` against actual lifecycle events. This is the highest-risk area for "id transition silently missed" bugs.

## Verdict

`merge-after-nits`

## Rationale

Protocol-additive change with the correct distillation pattern (new notification + watch channel + initialize-time replay). The exhaustive-match no-op on the TUI side is the right way to keep an unrelated consumer compiling. Six-surface schema symmetry is the load-bearing engineering invariant in this repo and the PR holds it. Nits: (1) verify a test pins all four publication trigger points (load / create / clear-for-account-change / backend-invalidate) — the PR body lists them but I want to see four named test cases, not a single "happy path" test; (2) verify the initialize-replay path explicitly skips sending when the current value is `None` (the PR body says "current non-null id" but the diff snippet I reviewed doesn't show the `if let Some(id)` guard); (3) consider adding a TUI consumer in a follow-up — a v2 notification with no in-tree consumer beyond an exhaustive-match no-op is a smell that the contract is under-tested end-to-end. None block merge.

