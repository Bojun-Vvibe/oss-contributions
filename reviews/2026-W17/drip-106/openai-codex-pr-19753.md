# openai/codex PR #19753 — Terminate stdio MCP servers on shutdown

- **Link**: https://github.com/openai/codex/pull/19753
- **Head SHA**: `d663dea0e1e39dd82824fce24027d0d91cb54abf`
- **Author**: etraut-openai
- **Size**: +356 / -32 across 8 files
- **Verdict**: `merge-as-is`

## Files changed
- `codex-rs/codex-mcp/src/connection_manager.rs` — adds `startup_cancellation_token: CancellationToken` to `McpConnectionManager`; adds `begin_shutdown(&mut self) -> impl Future<Output=()> + Send + 'static` that drains `clients`, clears `server_origins`, and returns a future that calls `client.shutdown().await` for each drained client; adds `shutdown(&mut self)` convenience wrapper; impls `Drop` that cancels the token + clears clients as a last-resort sweep.
- `codex-rs/codex-mcp/src/connection_manager_tests.rs` — five existing tests updated with the new `cancel_token: CancellationToken::new()` field (lines ~662, 691, 728, 753, 788).
- `codex-rs/codex-mcp/src/rmcp_client.rs` — `AsyncManagedClient` gets a `pub(crate) cancel_token: CancellationToken` field; the `start_server_task` future is restructured so the `or_cancel(&cancel_token_for_fut)` wrapper sits outside the `match` instead of inside, ensuring the cancel signal interrupts the entire startup pipeline (previously it only interrupted `start_server_task` itself).
- `codex-rs/core/src/connectors.rs`, `codex-rs/core/src/session/handlers.rs`, `codex-rs/core/src/session/mcp.rs` — call sites that own the connection manager now invoke `.shutdown().await` (or `.begin_shutdown()` and await later) at the appropriate session-end seam.
- `codex-rs/rmcp-client/src/bin/test_stdio_server.rs` (new) — long-running test stdio MCP server with PID-file emission and a `sync` tool that sleeps for the requested ms.
- `codex-rs/rmcp-client/src/rmcp_client.rs` — implementation of `client.shutdown()` that signals the cancel token AND waits for the server process to actually exit (PID polling via `process_exists`).

## Analysis

This is a textbook regression-coverage fix: the bug is "stdio MCP server processes survive thread shutdown", reported by four separate issues (#12491, #12976, #18881, #19469). The PR's `## History` section correctly diagnoses the regression — #10710 added process-group cleanup, but the session-level shutdown path never explicitly drove that cleanup, so the only thing terminating the children was the parent dying. For long-lived parent processes (the codex CLI staying open across multiple sessions), that meant per-session leaked stdio MCP processes.

The architecture is the right shape:

1. **Token, not signal**: a `CancellationToken` stored in the manager AND propagated into `AsyncManagedClient.cancel_token` is the right primitive for "stop in-flight startup AND stop fully-started clients". The relocation of `or_cancel(&cancel_token_for_fut)` in `rmcp_client.rs:144-185` from inside the inner `start_server_task` block to outside the entire `async { ... }` block is subtle but important — it ensures cancellation aborts the **server-process-spawn** stage too, not just the post-spawn task registration. Without that change, a `shutdown()` issued during the narrow window between `process::Command::spawn()` and `start_server_task` returning would leak.

2. **Two-phase shutdown via `begin_shutdown(&mut self) -> impl Future + 'static`**: the `'static` bound on the returned future is exactly the right contract for callers that want to drop the manager early but await termination later (e.g., `tokio::spawn(manager.begin_shutdown())` for fire-and-forget) versus callers that want to block until children exit (`manager.shutdown().await`). The drain-then-clear pattern (`std::mem::take(&mut self.clients)`, `self.server_origins.clear()`) inside `begin_shutdown` makes the manager immediately appear empty to the rest of the world even before children exit, which avoids races where a concurrent reader sees half-shutdown clients.

3. **`Drop` impl as a last-resort sweep** (lines ~633-638) is defensive and correct: cancels the startup token and clears clients. It can't `await` anything, so it can't actually wait for child processes to die — but combined with the cancel token signal and the `Send + 'static` futures, the runtime will run the cleanup. The combination of "explicit shutdown is preferred but Drop doesn't leak the token" is the right belt-and-suspenders.

4. **The integration test in the diff tail** is the highlight: it spawns a real test stdio server that writes its PID to a file, asserts `process_exists(server_pid)` BEFORE shutdown, dispatches a long-running `call_tool` with `sleep_after_ms: 300_000` so we're testing shutdown *during an active call*, calls `client.shutdown().await`, then asserts `wait_for_process_exit(server_pid)` and that the call task's join completes within 5s. That's the real bug the issues are reporting and the test pins it end-to-end.

## Verdict rationale

This is `merge-as-is` because: (a) the architecture matches the bug, (b) the cancel-token relocation in `rmcp_client.rs` fixes a real race nobody was asking about (which is a sign the author understood the lifecycle deeply), (c) the integration test exercises the actual leak path with a real OS process and PID polling, (d) `Drop` is correctly defensive without relying on async, (e) the call-site updates in `core/src/session/*.rs` are mechanical and follow the same `.shutdown().await` pattern.

## Optional follow-ups (not blocking)

- The PID-polling helper `wait_for_process_exit` should have a configurable timeout in test code for CI-flake tolerance — consider a 30s default with a `WAIT_FOR_PROCESS_EXIT_TIMEOUT_MS` env override.
- For platforms where `process_exists` returns true for zombie processes (Linux pre-`waitpid`), the test could spuriously time out if the spawn machinery doesn't reap. Worth a comment naming the assumption.
- The `begin_shutdown` future captures `clients` by value but the children are torn down sequentially via `for client in clients.into_values() { client.shutdown().await; }`. For workspaces with many MCP servers, parallelizing with `futures::future::join_all` would shorten shutdown time. Not a correctness issue, just a latency win.

## What I learned

The "rust async cleanup is hard" theme keeps coming up: `Drop` can't await, cancellation tokens have to be plumbed through every layer that owns a child task, and the seam between "drain the registry" and "wait for the children to exit" needs to be expressed as a returned `'static` future so callers can choose their fire-and-forget vs. wait-for-it semantics. This PR gets all three right and pins it with a real-process integration test — that's the gold standard for this class of fix.
