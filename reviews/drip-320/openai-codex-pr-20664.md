# openai/codex PR #20664 — Add stdio exec-server client transport

- Link: https://github.com/openai/codex/pull/20664
- SHA: `751ed42d78804323aca8cfede62afdec11a5ce29`
- Author: starr-openai
- Stats: +261 / −51, 8 files

## Summary

PR 2 of a 5-PR stack adding a stdio JSON-RPC transport to the exec-server client. Previously the only way for Codex to talk to an exec-server was via a websocket URL. This change introduces a `StdioExecServerConnectArgs` variant and an `ExecServerTransport` enum so a configured environment can spawn an exec-server child process and speak JSON-RPC over its stdio, with the child's lifetime bound to the client.

The refactor pushes the transport choice up into `LazyRemoteExecServerClient` (which now stores an `ExecServerTransport` instead of a raw websocket URL) and exposes a generic `connect_for_environment()` method on the transport. The old `ExecServerClient::connect_websocket` helper is removed in favour of routing everything through `ExecServerClient::connect(JsonRpcConnection, ExecServerClientConnectOptions)`, which is now `pub(crate)`.

## Specific references

- `codex-rs/exec-server/src/client.rs` L19–L26: imports for `ExecServerTransport` and `StdioExecServerConnectArgs` from `client_api`. Layering looks clean — the client module no longer reaches into `tokio_tungstenite` directly.
- `codex-rs/exec-server/src/client.rs` L108–L116: the `From<StdioExecServerConnectArgs> for ExecServerClientConnectOptions` impl mirrors the existing `From<RemoteExecServerConnectArgs>` impl. Good symmetry. Worth confirming the two arg structs agree on which fields are required vs optional (e.g. `connect_timeout` exists on the websocket variant but presumably not on stdio because process spawn timeout lives elsewhere).
- `codex-rs/exec-server/src/client.rs` L193–L213: `LazyRemoteExecServerClient` now stores `transport: ExecServerTransport`. The naming is slightly stale — "LazyRemoteExecServerClient" implies remote/network, but it now also wraps a local stdio transport. Consider renaming to `LazyExecServerClient` in a follow-up.
- `codex-rs/exec-server/src/client.rs` L262–L286: removal of `connect_websocket`. Any external (non-`pub(crate)`) callers of `connect_websocket` would break — confirm there are none in `codex-rs/` workspace; the diff implies all call sites now go through `transport.connect_for_environment()`.
- `codex-rs/exec-server/src/client.rs` L410–L414: `connect` is widened from private to `pub(crate)`. Justified by the new `client_transport.rs` needing it; confirm no external crate depends on the old visibility.
- `codex-rs/exec-server/src/client_api.rs`, `client_transport.rs`, `connection.rs`, `environment.rs`, `lib.rs`, `rpc.rs`, `server/processor.rs`: not inspected line-by-line in this drip; the shape implied by the imports suggests `ExecServerTransport` is a sum type with `Websocket(RemoteExecServerConnectArgs)` and `Stdio(StdioExecServerConnectArgs)` variants, which is the right modelling.

## Verdict

verdict: merge-after-nits

## Reasoning

Solid refactor that generalises the transport without disturbing existing websocket callers. Pre-merge nits: rename `LazyRemoteExecServerClient` (or document why "Remote" stays), and verify all old `connect_websocket` callers in the workspace have been migrated. As PR 2 of a 5-stack it makes more sense to land as-is and address the rename in a later cleanup pass within the stack.
