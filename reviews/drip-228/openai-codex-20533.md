# openai/codex #20533 — Add exec-server status endpoints

- **PR**: https://github.com/openai/codex/pull/20533
- **Head SHA**: `519997aa2081ba8f6e839aeb60d28f1eb612742f`
- **Files reviewed**:
  - `codex-rs/exec-server/Cargo.toml` (+6 axum dep)
  - `codex-rs/exec-server/README.md` (+16 endpoint docs)
  - `codex-rs/exec-server/src/connection.rs` (+135 axum-WebSocket adapter)
  - `codex-rs/exec-server/src/local_process.rs` (+39 -2 status counter wiring)
  - `codex-rs/exec-server/src/server.rs` (+1 module decl)
  - `codex-rs/exec-server/src/server/process_handler.rs` (+13 -2)
  - `codex-rs/exec-server/src/server/handler/tests.rs` (+4 -4 `new_for_tests`)
  - `codex-rs/exec-server/BUILD.bazel` (+1 axum dep_extra)
- **Date**: 2026-05-01 (drip-228)

## Context

`codex exec-server --listen ws://...` previously exposed a single
WebSocket endpoint and nothing else. That's a dead-end for ops:
no liveness probe for k8s, no readiness probe for gradual rollouts,
no Prometheus scrape target, no human-readable status page. This PR
adds `/healthz`, `/readyz`, `/status`, `/metrics` to the same TCP
listener while preserving the documented stdout contract (the
WebSocket URL remains the only stdout startup line).

## Diff walk

**`README.md:30-46`** — explicit operator-facing contract:
- `/healthz` → `200 OK / ok\n` when the process can answer
- `/readyz` → `200 OK / ready\n` when required helper paths are usable, else `503 / not ready\n`
- `/status` → JSON summary, intentionally **omits command lines, env
  vars, working dirs, session ids, pids, user names, local paths**
  (the privacy-preservation list is named in the README — that's the
  contract a future contributor will be evaluated against)
- `/metrics` → low-cardinality Prometheus text: uptime, connections,
  sessions, processes, JSON-RPC request totals by **fixed method name
  and result** (the "fixed" qualifier is the load-bearing word — it
  rules out per-session/per-pid label explosion)

**`connection.rs:37-172`** — new `JsonRpcConnection::from_axum_websocket(stream, label)`
constructor that mirrors the existing `from_stdio` shape: spawns a
reader task and a writer task over the axum WebSocket primitives,
sends `JsonRpcConnectionEvent::Message` for parsed JSON-RPC frames,
sends `Disconnected{reason}` on close/IO-error, and the reader task
explicitly handles all four `AxumWebSocketMessage` variants
(`Text`/`Binary`/`Close`/`Ping`/`Pong`) — the Ping/Pong arm is `{}`
which is correct (axum auto-pongs at the protocol layer).

**`local_process.rs:88, 110-122, 268`** — threads
`Arc<ExecServerStatusState>` into `LocalProcess::new`,
adds a `status_snapshot()` async method that locks the process map
and counts `Starting`/`Running`/`exited_retained` entries, and calls
`self.inner.status.process_started()` after each successful spawn at
`:266`. The `Default` impl at `:106-122` panics with explicit
diagnostic strings (`"current executable should resolve: {err}"`) if
the binary path resolution fails — that's correct for the Default impl
because Default is only used in tests / fallback paths where panicking
beats silently mis-configuring the snapshot.

**`server/processor.rs`** — wires the status state through the request
processor so per-method counters fire on every JSON-RPC dispatch. The
sample shows `use crate::server::status::ExecServerStatusState;`
imported alongside the existing handler/registry/session_registry
modules — the status module is sibling-scoped and `pub(crate)` only.

**`handler/tests.rs:80, 158, 231, 275`** — four call-site updates from
`SessionRegistry::new()` to `SessionRegistry::new_for_tests()`. The
constructor split is the right shape: production callers get the
status-aware constructor, tests get a no-status variant that doesn't
require building an `ExecServerStatusState` just to exercise handler
logic.

## Observations

1. **The privacy-preservation contract is enumerated, not implied.**
   The README explicitly names the eight things `/status` must not
   expose. That makes the contract testable — a follow-up linter or
   review could grep the JSON serializer for any reference to
   `command_line`/`env`/`cwd`/`session_id`/`pid`/`uid`/`user_name` and
   fail. Compare to the alternative ("status returns a sensible
   summary") which would inevitably grow PII over time.

2. **Cardinality discipline on `/metrics` is documented up front.**
   "Low-cardinality" + "fixed method name and result" is the right
   shape for Prometheus — it rules out per-session/per-pid label
   explosion that's the #1 way Prometheus exposition pages cause
   incidents. A future PR adding a per-`(client_id, method, status)`
   counter would now visibly violate the README contract.

3. **`from_axum_websocket` symmetry with `from_stdio` is real, not
   cosmetic.** Both constructors return the same `JsonRpcConnection`
   shape, both spawn a reader task that pushes
   `JsonRpcConnectionEvent` into the same channel, both spawn a writer
   task that drains `outgoing_rx`. That means the upstream
   message-processing loop in the handler has zero awareness of which
   transport it's on — a strong abstraction boundary.

4. **Nit: malformed-message handling on the WebSocket reader doesn't
   close the connection.** When `serde_json::from_str` fails at
   `:48-58`, the reader sends `MalformedMessage{reason}` and **keeps
   reading**. That matches the stdio behavior (presumably) but for a
   WebSocket peer this means a malicious or buggy client can keep
   blasting unparseable frames forever. Worth a doc-line in
   `from_axum_websocket` explaining the deliberate choice, or a
   counter in `/metrics` so an operator can detect the pattern. Not
   blocking.

5. **`Default` panic strings are diagnostic-grade.** `"current
   executable should resolve: {err}"` and `"current executable should
   be absolute: {err}"` are exactly the right level of detail —
   names what was tried, includes the underlying error, doesn't
   over-explain. If the panic ever fires in the wild the operator
   knows where to look.

6. **No load test in the PR.** Adding four HTTP routes to the same
   listener as the WebSocket means a slow `/status` response could
   theoretically backpressure WebSocket accept. The diff doesn't
   include a smoke test asserting the routes co-exist under
   concurrent WebSocket connect + HTTP scrape. Worth one
   integration test — even an `axum_test` round-trip — to lock in
   the "they share a TCP listener" contract.

## Verdict

**merge-after-nits** — the operational contract is precisely written,
the WebSocket adapter is symmetric with the stdio adapter, the
privacy and cardinality discipline is documented at the README
boundary. Add (a) a co-existence smoke test for the HTTP routes
alongside a live WebSocket and (b) one line about the
malformed-frame-doesn't-close-WS deliberate choice and ship.

## What I learned

When adding ops endpoints to a previously single-purpose listener,
the README is the load-bearing artifact, not the code. The eight
explicitly-banned `/status` fields and the "fixed labels only"
`/metrics` policy create a contract that will outlive the people who
wrote it — which is exactly the property you want for the parts of
the system that ops teams build dashboards against.
