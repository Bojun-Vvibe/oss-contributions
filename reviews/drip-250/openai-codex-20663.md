# openai/codex #20663 — Add stdio exec-server listener

- URL: https://github.com/openai/codex/pull/20663
- Head SHA: `e2fda326ea5706184320ae649b49bae52fa67a7d`
- Files: `codex-rs/cli/src/main.rs` (+1/-1), `codex-rs/exec-server/src/connection.rs` (+0/-6), `codex-rs/exec-server/src/server/transport.rs` (+40/-7), `codex-rs/exec-server/src/server/transport_tests.rs` (+28/-12)

## Context

First in a 5-PR stack splitting up the original draft #20508 — adds `stdio` and `stdio://` as accepted `--listen` transports for the exec-server, alongside the existing `ws://IP:PORT`. Server-side only: this PR doesn't ship a client that spawns the server over stdio (#20664 does that), no env config, no product wiring. Just: "the server can now answer JSON-RPC on stdin/stdout if you ask it to."

## Design analysis

The shape is right and minimal:

1. **Typed transport enum** at `transport.rs:14-18`:
   ```rust
   pub(crate) enum ExecServerListenTransport {
       WebSocket(SocketAddr),
       Stdio,
   }
   ```
   replaces the prior `parse_listen_url(&str) -> Result<SocketAddr, _>` return shape — which couldn't represent "stdio" at all, since stdio has no socket address. The dispatch `match parse_listen_url(listen_url)? { WebSocket(addr) => run_websocket_listener(addr, ...), Stdio => run_stdio_connection(...) }` at `:67-72` is the canonical typed-fork.

2. **Single-client semantics for stdio is implicit and correct**. `run_stdio_connection` (`:78-90`) instantiates one `ConnectionProcessor` and runs `JsonRpcConnection::from_stdio(io::stdin(), io::stdout(), "exec-server stdio".to_string())` to completion — no listener loop, no fan-out. That's the right shape for stdio (you can't accept-loop on stdin), and the parent caller (`run_transport`) just awaits the single connection. No risk of "second client connects and kills the first" because there is no accept loop.

3. **Promotion of `from_stdio` out of `#[cfg(test)]`** at `connection.rs:11-15`, `:31-32`, `:296`. The previously test-only helper `JsonRpcConnection::from_stdio<R, W>` and `write_jsonrpc_line_message` are now production code. The implementation didn't change; only the visibility gate. Worth checking: the function takes `R: AsyncRead + Unpin + Send + 'static, W: AsyncWrite + Unpin + Send + 'static` which matches `tokio::io::Stdin` / `tokio::io::Stdout`. That's the right contract for production stdio, and the tests already exercise it.

4. **Parser accepts both spellings** at `transport.rs:48-50`:
   ```rust
   if matches!(listen_url, "stdio" | "stdio://") {
       return Ok(ExecServerListenTransport::Stdio);
   }
   ```
   The bare `stdio` form is conventional (matches MCP's `stdio` transport spelling), the `stdio://` URL form keeps lexical consistency with `ws://...`. Tests `parse_listen_url_accepts_stdio` and `parse_listen_url_accepts_stdio_url` at `transport_tests.rs:24-35` lock both spellings. Error message updated at `:64`: `expected `ws://IP:PORT` or `stdio``.

## Risks

- **`tracing::info!("codex-exec-server listening on stdio")`** at `transport.rs:84` — tracing output by default goes to stderr in this codebase (need to confirm), which is fine for stdio mode. If tracing is ever reconfigured to land on stdout, it will corrupt the JSON-RPC stream. Worth a defensive comment near the stdio handler: "do not write any non-JSON-RPC bytes to stdout on this transport." A debug-only assert that the tracing subscriber writer is not stdout would be paranoid but cheap.
- **No graceful shutdown signal** — the WebSocket path can be terminated by closing the listener; the stdio path runs until `JsonRpcConnection::from_stdio` returns (i.e. EOF on stdin, or a JSON-RPC `shutdown` notification). Neither is wired to a SIGTERM handler in this PR. Probably fine for a single-connection process, but worth confirming in stack PRs #20664+ that the parent (the spawning client) closes stdin to signal shutdown.
- **Transport identity in logs** — the connection label `"exec-server stdio".to_string()` is a fixed string; the WebSocket path presumably labels each connection with its peer addr. For debugging multiple stdio invocations across `tmux` panes, a PID or process-start timestamp in the label would help — but this is the right kind of nit to defer until the client side (#20664) lands.

## Suggestions

- Defensive comment at the top of `run_stdio_connection` warning about stdout protocol purity.
- `parse_listen_url_rejects_stdio_with_garbage` test for `stdio://garbage` — the current code accepts only `stdio` and `stdio://` exactly; a `stdio://path/that/looks/structured` will fall through to the `UnsupportedListenUrl` arm. Locking this is a 5-line test that prevents future accidental loosening.
- Update `cli/src/main.rs:449` doc comment from `Supported values: \`ws://IP:PORT\` (default), \`stdio\`, \`stdio://\`.` to also note "stdio runs a single in-process JSON-RPC connection over stdin/stdout and exits on EOF" — sets correct mental model for users typing `--listen stdio`.

## Verdict

`merge-after-nits` — clean, narrowly-scoped first slice of the 5-PR stack: typed transport enum replaces a `SocketAddr`-only return, dispatch is a two-arm match, stdio handler is a single-connection-to-completion shape that's the right model for stdin/stdout, both `stdio` and `stdio://` spellings tested, error message updated. Wants stdout-purity defensive comment and a stdio-with-garbage rejection test. The pre-existing `from_stdio` helper graduating from test-only to production is the right factoring.
