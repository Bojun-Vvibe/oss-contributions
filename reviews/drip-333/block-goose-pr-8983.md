# block/goose PR #8983 — fix: SACP notifies clients of generated session names

- **Link:** https://github.com/block/goose/pull/8983
- **Head SHA:** `6cab656232064992915444579d1b5f4b77999863`
- **Verdict:** `merge-after-nits`

## What it does

Wires up an `acp` `SessionInfoUpdate` notification path so that
auto-generated session names actually reach connected SACP clients. Three
load-bearing changes:

1. **New mpsc channel + spawned forwarder** at
   `crates/goose/src/acp/server.rs:271-301`:
   `spawn_session_name_update_notifier(cx)` returns an
   `UnboundedSender<SessionNameUpdate>` whose receiver loop builds a
   `SessionNotification` carrying `SessionUpdate::SessionInfoUpdate(...)`
   and forwards via `cx.send_notification(...)`.
2. **`Agent::maybe_update_name` return-type change** at
   `crates/goose/src/session/session_manager.rs:359-389`: now returns
   `Result<Option<SessionNameUpdate>>` instead of `Result<()>`. The
   `Some` branch is built only when (a) the user hasn't set the name and
   (b) the new name actually differs from the existing one
   (`thread.user_set_name && thread.name != name` at line 392).
3. **`AgentConfig::session_name_update_tx`** at
   `crates/goose/src/agents/agent.rs:118` plus builder
   `with_session_name_update_tx` at `:148-154`. The agent's spawned
   `maybe_update_name` task at `agent.rs:1257-1273` now matches on the
   returned `Option<SessionNameUpdate>` and forwards via the channel
   when present.

Also fixes a parallel hole in the explicit `rename_session` request
handler at `acp/server/sessions.rs:148-163` so a user-initiated rename
also writes through to `session_manager.update(...).user_provided_name(title)`,
not just to the `thread_manager`.

## Design analysis

- **Channel-based forwarding is the right primitive.** The session-name
  generator runs in a background `tokio::spawn` (`agent.rs:1259`) detached
  from the request that triggered it, so it cannot directly hold a
  reference to the caller's `cx: ConnectionTo<Client>`. An mpsc channel
  is the canonical decoupling — the agent owns the sender (cheap to
  clone), the spawned forwarder owns the receiver and the connection
  handle. Drop semantics on the sender naturally tear down the forwarder
  when the agent goes away.
- **`Option<SessionNameUpdate>` with the "no-change" early return** at
  `session_manager.rs:392` is the load-bearing detail. Without the
  `&& thread.name != name` check, every agent turn that re-runs the
  naming pipeline would emit a redundant SessionInfoUpdate notification
  even when the name didn't change — UI clients would see flicker.
- **`spawn_session_name_update_notifier` is gated behind
  `(!disable_session_naming).then(...)`** at `server.rs:1187-1188`. If
  naming is disabled, no channel exists, no task spawns. Good resource
  hygiene — the forwarder isn't sitting idle on a never-firing receiver.
- **`SessionInfoUpdate.title(...).updated_at(...).meta(...)` builder
  chain** at `server.rs:284-289` carries enough information for clients
  to update their local session list without re-fetching. The `meta` is
  built via the existing `thread_session_meta(&thread)` helper so the
  shape stays consistent with other places that emit session info.

## Tests

`crates/goose/tests/acp_common_tests/mod.rs:130-180` adds
`run_session_name_update_notification` exercising the full path: a
fixture `NamingProvider` (lines 61-95) that returns `"Generated Test
Title"` when its system prompt contains the four-words-or-less naming
hint, otherwise returns `"2"` for the prompt routing logic. The test
prompts the session, drains notifications with a 1-second deadline, and
asserts the forwarded `SessionInfoUpdate` carries the generated title.
`spawn_acp_server_in_process` signature at `acp_fixtures/mod.rs:160`
gains `disable_session_naming: bool` so the test can flip it on.

## Risks / nits

1. The mpsc is `unbounded_channel` at `server.rs:273`. Session-name
   generation fires once per session (or once per significant context
   change), so unbounded is fine in steady state, but if a misbehaving
   provider were to hammer name regeneration the channel would grow
   unbounded. Worth a `bounded(64)` or similar with a `try_send` and
   warn-on-full path for defensive memory hygiene.
2. The forwarder's `warn!(...)` on send failure at `server.rs:296-300`
   is the right level, but doesn't include the title content — useful
   for debug purposes when a notification disappears.
3. `tx.send(update).is_err()` at `agent.rs:1267` is checked but the
   error case (`warn!("Failed to publish generated session name")`)
   doesn't carry the session id or title for forensic value. Two
   places to log degraded behavior — either consolidate or enrich.
4. `spawn_session_name_update_notifier` clones `cx` at the call site
   `server.rs:1188`. `ConnectionTo<Client>` is presumably cheap-clone
   (likely an `Arc` or similar), but worth a doc comment confirming.
5. The `&& thread.name != name` check at
   `session_manager.rs:392` does a string equality on potentially
   large names. In practice names are <80 chars so this is a non-issue,
   but if names grow into multi-paragraph descriptions it'd be worth
   hashing.
6. The test at `acp_common_tests/mod.rs:163-180` uses a 1-second
   deadline polling at 10ms intervals (100 polls). Reasonable for CI
   under load but could flake if the provider stream takes >1s. A
   tokio `Notify`-based wait would be more robust than polling.

## What I learned

The "background task generates a name, but the request that triggered
it has already returned" pattern is universal for ACP-style streaming
protocols. The right shape is *always* a channel from the background
task to a long-lived forwarder that owns the connection — not threading
the connection through the background task itself, and not making the
caller wait. The Option-returning "name actually changed" early-out is
the secondary load-bearing detail to prevent notification spam, and
this PR gets both primitives right.
