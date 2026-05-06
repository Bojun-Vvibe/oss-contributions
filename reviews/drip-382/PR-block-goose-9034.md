# block/goose PR #9034 — feat: ACP streamable http spec compliance

- URL: https://github.com/block/goose/pull/9034
- Head SHA: `289ae524573e06e941b0e281e8ad8822557e7c5e`
- Size: +420 / -167

## Summary

Updates goose's reference ACP (Agent Client Protocol) HTTP/WS transport
implementation — both server (`crates/goose/src/acp/transport/`) and JS SDK
client (`ui/sdk/src/http-stream.ts`) — to align with the latest
streamable-http-websocket-transport RFD revision (referenced as ACP RFD
PR #1124 in the description). Reorganizes the connection-level fan-out
from a single broadcast channel into three coexisting streams: a
*connection-scoped* stream, a *per-session* stream keyed by `sessionId`,
and an *all-outbound* stream that the WebSocket transport multicasts on.
Adds a JSON-RPC ID → `ResponseRoute` (`Connection` | `Session(sid)`)
pending-routes table so server-side responses to client-initiated requests
land on the same scope the request came in on.

## Specific findings

- `crates/goose/src/acp/transport/connection.rs:24-29` — new
  `ResponseRoute { Connection, Session(String) }` enum. Correct
  granularity: the ACP wire spec scopes session notifications by
  `params.sessionId`, and HTTP `GET /acp/sessions/{id}` SSE consumers need
  to receive *only* messages for their session, while a connection-scoped
  GET stream should see only non-session-scoped traffic.
- `connection.rs:32-79` — `OutboundStream` extracts the previous
  `(broadcast::Sender<String>, Mutex<Option<VecDeque<String>>>)` pair
  into a reusable struct with `push()` / `subscribe_with_replay()`. The
  pre-subscribe buffer behavior (drop-oldest at
  `PRE_SUBSCRIBE_BUFFER_CAPACITY = 1024` with a `warn!`) is preserved
  verbatim from the old implementation. Three independent
  `Arc<OutboundStream>` instances — `connection_stream`, per-entry of
  `session_streams: Arc<RwLock<HashMap<String, Arc<OutboundStream>>>>`,
  and `all_outbound` — coexist on each `Connection`.
- `connection.rs:163-176` — `route_outbound` is the new fan-out core.
  Every outbound message is *unconditionally* pushed to `all_outbound`
  first (so WebSocket clients see everything), then classified and
  pushed to either `connection_stream` or the appropriate
  `session_streams[sid]`. The `serde_json::from_str(&msg).ok()` fallback
  to `Target::Connection` on parse failure is correct: malformed
  outbound JSON shouldn't be silently dropped, and the connection scope
  is the right default for an unparseable server-emitted line.
- `connection.rs:180-204` — `classify(v)` decision tree:
  1. has `method` and a session id in `params.sessionId` → session stream;
  2. has `method` but no session id → connection stream;
  3. has `id` + (`result` | `error`) → look up `pending_routes[id]`,
     defaulting to connection on miss;
  4. else → connection.
  Branch 3's `pending_routes.lock().await.remove(&id)` correctly cleans
  the entry on use, preventing the table from growing unbounded across a
  long session.
- `connection.rs:206-211` — `record_pending_route` short-circuits
  `id.is_null()` so notification responses (which JSON-RPC permits with
  null id, though uncommon for these flows) don't accumulate dead
  entries. Correct guard.
- `connection.rs:108-128` — `Connection` constructor wires the three
  streams plus the `pending_routes` mutex; `agent_handle` /
  `router_handle` shutdown semantics retained from the old
  `pump_handle`. `shutdown()` at `connection.rs:262-268` aborts both.
- `ui/sdk/src/http-stream.ts:+126/-39` — JS SDK side updated to consume
  the new per-session SSE topology. (Diff snippet not fully captured in
  this review pass; reviewer should spot-check that the SDK `EventSource`
  URL and reconnect logic match the new server route shape.)

## Notes

- `pending_routes` is keyed by `serde_json::Value` (the raw JSON-RPC id).
  JSON-RPC ids can be string, number, or null. Number id `42` and string
  id `"42"` are distinct keys here, which is correct per spec but
  brittle if a client ever round-trips an id through a string-stringifying
  middleware. Worth a one-line comment naming the assumption.
- No new test file exercising the routing classifier in isolation.
  Verification in the PR body is a single end-to-end smoke
  (`cargo run --bin goose -- serve --port 3000` + `npm start ...
  --text "say hi in 3 words"`). For a 420-line transport rewrite this is
  thin — at minimum a Rust-side unit test that feeds a hand-rolled
  `Value` through `classify` and asserts each of the four branches would
  pin the contract.
- `OUTBOUND_BROADCAST_CAPACITY = 1024` is now applied per
  `OutboundStream`, not per `Connection`. With one connection-scoped
  stream + N per-session streams + one all-outbound stream, total
  per-connection broadcast capacity is `(N+2) × 1024`, which under load
  with many concurrent sessions could be a memory surprise. Probably
  fine for typical agent workflows, but worth a comment.
- `route_outbound` clones `msg` once (for `all_outbound.push(msg.clone())`)
  before the classify branch. For a typical short notification this is
  fine; for very large image-payload messages it doubles the heap
  pressure. If the per-stream broadcast already shares an `Arc<String>`
  internally, this clone is cheap; if not, an `Arc<str>`-typed channel
  would be a future optimization.
- The ACP RFD link in the PR body
  (`agentclientprotocol/agent-client-protocol/blob/main/docs/rfds/streamable-http-websocket-transport.mdx`)
  is the canonical contract. Reviewer should diff the message-routing
  rules in the RFD against `classify()` to confirm spec parity.
- No CHANGELOG entry for what is a wire-protocol-shape change. Worth
  adding before merge.

## Verdict

`needs-discussion`
