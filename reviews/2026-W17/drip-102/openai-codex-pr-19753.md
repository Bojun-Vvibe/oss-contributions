# openai/codex PR #19753 — Terminate stdio MCP servers on shutdown to avoid process leaks

- Link: https://github.com/openai/codex/pull/19753
- Head SHA: `d663dea0e1e39dd82824fce24027d0d91cb54abf`
- Size: +356 / -32 across 10 files

## Summary

Closes a long-standing MCP lifecycle gap (#12491, #12976, #18881, #19469) where stdio MCP server processes survived session shutdown. PR #10710 had earlier added process-group cleanup, but it only fired on `RmcpClient`/transport `Drop` — and several higher-level paths (session shutdown via the explicit `Op::Shutdown`, channel-close fallback when the submission loop exits without an explicit shutdown op, MCP refresh, connector probing) kept the manager alive or replaced it without ever draining clients first. This PR adds explicit `shutdown()` / `begin_shutdown()` on `McpConnectionManager`, an `AsyncManagedClient::shutdown()` that cancels the startup token and drains the underlying client, and an explicit stdio process-handle terminate path so cleanup no longer depends on `Drop` ordering.

## Specific-line citations

- `codex-mcp/src/connection_manager.rs:71-110` adds a `startup_cancellation_token: CancellationToken` field, the `begin_shutdown()` method that cancels the token + `mem::take`s the `clients` HashMap + clears `server_origins` and returns a `'static + Send` future that drives the actual `client.shutdown().await` calls (the take-and-return-future shape is correct because it lets the caller release the manager's write lock before awaiting the per-client shutdown), and a thin `pub async fn shutdown()` wrapper.
- `connection_manager.rs:633-639` adds a `Drop` impl that cancels the startup token + clears `clients` so an unexpected drop still cancels in-flight startups, even if it can't await the per-client shutdown.
- `codex-rmcp-client` introduces `terminated: AtomicBool` (`:495`), changes `_process_group_guard: Option<ProcessGroupGuard>` (which only cleaned up on Drop, `:442/:521`) to an explicit `process: ProcessHandle` with `process.terminate().await?` at `:451`, and renames `maybe_terminate_process_group` → `terminate` at `:549-552` — so the cleanup is now a deliberate call rather than an accident of Drop timing.
- `core/src/session/handlers.rs:943-948` is the explicit `Op::Shutdown` path: takes the `mcp_connection_manager.write().await` lock, calls `manager.begin_shutdown()` to capture the future, **drops the lock** (the future is `'static`), then awaits the future. This is the right ordering — holding the write lock across an await on N stdio shutdowns would deadlock anything else trying to read the manager.
- `handlers.rs:1236-1252` is the implicit channel-close path: same shape, plus an explicit `unified_exec_manager.terminate_all_processes()` call before, and an updated comment explaining "if the submission loop exits because the channel closed without an explicit shutdown op, still run process teardown" — closing the regression class identified in #18881.
- `connectors.rs:270/:349` makes the previously-leaked `mcp_connection_manager` from `list_accessible_connectors_from_mcp_tools_with_environment_manager` mutable and calls `mcp_connection_manager.shutdown().await` on the success path — connector probing was creating a manager, returning the result, and dropping it with no shutdown, which was the regression class of #19469.
- `AsyncManagedClient::shutdown` at `:208-217` correctly handles the in-flight startup case: cancels the token, then `match self.client().await` returns `StartupOutcomeError::Cancelled` (no-op, expected) vs other init errors (warn) vs Ok (`client.shutdown().await`).

## Verdict

**merge-as-is**

## Rationale

This is exactly the right shape for an MCP lifecycle fix. Three things make it merge-as-is rather than merge-after-nits:

1. The `begin_shutdown()` returning a `'static + Send` future is the correct pattern for "release this lock, then do potentially-slow IO" — anyone reviewing the locking story can confirm by inspection that no `.write().await` lock is held across the per-client shutdown awaits.
2. Replacing `_process_group_guard: Option<ProcessGroupGuard>` (Drop-based, leading-underscore "we know it's unused but holding it for RAII") with `process: ProcessHandle` + explicit `terminate()` is the right refactor — it converts an implicit ordering constraint into a callable, which is what makes the "manager replaced without draining" class of bug expressible as a missing call rather than an invisible Drop ordering issue.
3. The four issue numbers cited in the PR body (`#12491`, `#12976`, `#18881`, `#19469`) each map to a distinct call site that this PR touches: `Op::Shutdown` (`handlers.rs:943`), channel-close fallback (`handlers.rs:1236`), MCP refresh (in `session/mcp.rs` per the diff trailer), and connector probing (`connectors.rs:349`). The fix surface and the bug-report surface match.

The PR body mentions "regression coverage that starts a stdio MCP server, begins an in-flight blocking tool call, shuts down" — that's the right test to pin the specific class. Worth confirming in the test file (`connection_manager_tests.rs:662-695` shows the new `cancel_token: CancellationToken::new()` field threading through existing tests, which is good test-shape evolution).
