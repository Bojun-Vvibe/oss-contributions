# openai/codex #20361 — realtime: rename provider session ids in bidi surfaces

- **Author:** aibrahim-oai (Ahmed Ibrahim)
- **SHA:** `e982c16`
- **State:** OPEN
- **Size:** +336 / -147 across the protocol crate, app-server, core realtime, TUI, and regenerated JSON/TS schemas
- **Verdict:** `merge-after-nits`

## Summary

The codex-side identifier-naming series tightens. `session_id` on the realtime
bidi/start surfaces is renamed to `realtime_session_id` (and `sessionId` →
`realtimeSessionId` on the wire) in `ConversationStartParams`,
`RealtimeConversationStartedEvent`, `RealtimeEvent::SessionUpdated`,
`v2::ThreadRealtimeStartParams`, and `v2::ThreadRealtimeStartedNotification`.
Wire backward-compat is preserved via a single `#[serde(alias = "sessionId")]`
on the start-params struct field at `app-server-protocol/src/protocol/v2.rs:5295`
plus a dedicated `deserialize_thread_realtime_start_accepts_legacy_session_id`
regression at `protocol/common.rs:2675-2705`. The README at
`app-server/README.md:761-771,1100-1103` is updated with the load-bearing
clarification "this `realtimeSessionId` value refers to the upstream Realtime
API session, not a Codex session/thread-group id."

## Reasoning

This is a name-disambiguation refactor in the run-up to "session" being
repurposed as "thread group" elsewhere in codex. Doing it now is correct — the
longer the ambiguity stands, the more downstream surfaces silently couple to
the wrong meaning. The shape is right:

1. **Wire compat is asymmetric** — input accepts both `sessionId` and
   `realtimeSessionId` (alias on the deserializer), but output only emits
   the new name. That's the standard "consumers update first, then
   producers" rename pattern, and the new
   `deserialize_thread_realtime_start_accepts_legacy_session_id` test at
   `protocol/common.rs:2675-2705` locks the legacy input acceptance.

2. **The notification (server → client) drops the old name in one shot** at
   `protocol/v2.rs:5387` with no alias — correct for a server-to-client
   notification because the producer side controls timing, but worth
   confirming there's a SDK-bump coordination story.

Three nits to address before merge:

1. **No deprecation timeline on the `sessionId` input alias.** The alias
   should carry a `// TODO(2026-Qx): drop after SDK bump to vY.Z` comment so
   it doesn't live forever — wire-compat aliases that nobody removes
   accumulate into a maintenance hazard.

2. **The `EventMsg::RealtimeConversationStarted` rename at
   `bespoke_event_handling.rs:387-391`** assumes the `event.realtime_session_id`
   field name on the inner core event has also been renamed in the same
   commit. Worth one explicit cross-crate search-confirmation — a partial
   rename here would compile but produce a `realtime_session_id: None` for
   every started notification.

3. **The TUI tests were skipped due to a local linker bug per the PR body.**
   The TUI realtime state object presumably also got renamed. Confirm the
   serializer/deserializer there isn't relying on `session_id` (would
   produce silent JSON-level drift between TUI persistence and the new
   notification wire shape) before merge.
