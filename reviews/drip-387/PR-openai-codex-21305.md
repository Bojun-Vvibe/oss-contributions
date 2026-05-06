# Review: openai/codex #21305 — [codex] Add exec-server SIGTERM shutdown

- Carrier: `openai/codex`
- PR: #21305
- Head SHA: `c968b85f`
- Drip: 387

## Verdict
`merge-after-nits` (man)

## Summary
Adds graceful SIGTERM handling to `codex-exec-server` for both the
stdio-launched and websocket-listener modes, with three coordinated
mechanisms:
1. A new `tokio_util::sync::CancellationToken` threaded through
   `ConnectionProcessor::run_connection` and into the per-connection loop
   so a single signal can fan out to every active handler.
2. A new `SessionRegistry::shutdown_all()` that drains the session map
   under the existing `Mutex`, then calls `process.shutdown()` on each
   entry sequentially.
3. A new `shutdown_signal()` helper that listens for `SignalKind::terminate`
   on Unix and futures-`pending()` on non-Unix (no-op on Windows where
   SIGTERM doesn't exist), returning `IoResult<()>`.

The websocket listener wraps the `accept()` loop in `tokio::select!` against
`shutdown_signal`, then on signal: cancels the per-connection token, closes
the `TaskTracker`, awaits all in-flight connections, and finally calls
`processor.shutdown()`. The stdio path is the simpler analogue: spawn a
detached signal-listener task whose only job is to cancel the shared token,
then `signal_task.abort()` + `processor.shutdown()` on natural exit.

## Diff anchors
- `codex-rs/exec-server/Cargo.toml:36-43` — adds `"signal"` to the tokio
  feature list. Minimal; correctly avoids `"full"`.
- `codex-rs/exec-server/src/server/processor.rs:32-58` —
  `run_connection(self, connection, shutdown_token)` signature change with
  the new `shutdown()` method on `ConnectionProcessor` that delegates to
  `session_registry.shutdown_all()`.
- `:85-103` — the per-connection `incoming_rx.recv()` loop is now wrapped
  in `tokio::select!` against `shutdown_token.cancelled()`. The pattern
  `let event = ...; let Some(event) = event else { break; }` correctly
  preserves the existing channel-close termination semantics.
- `:120-132` — both nested `tokio::select!` blocks (request handler at
  `:123` and notification handler at `:166`) gain a `shutdown_token.cancelled()`
  arm so an in-flight handler can be cancelled mid-await.
- `codex-rs/exec-server/src/server/session_registry.rs:119-128` —
  `shutdown_all()` correctly drains the `HashMap` inside the `Mutex` scope
  *before* awaiting per-session shutdowns, avoiding a deadlock where a
  shutdown task tries to remove its own entry.
- `codex-rs/exec-server/src/server/transport.rs:96-119` — stdio mode
  spawns the signal-listener as a detached task, runs the connection,
  then `signal_task.abort()` + `processor.shutdown()` on return.
- `:131-176` — websocket mode pins the future via `tokio::pin!` and uses
  `&mut shutdown_signal` in the select arm to allow re-polling. The
  `connection_tasks: TaskTracker` correctly wraps `tokio::spawn` so the
  outer shutdown can `close().wait().await` for in-flight WebSocket
  connections instead of orphaning them.
- `:185-212` — `shutdown_signal()` helper. Unix branch uses
  `signal(SignalKind::terminate())?` then `term.recv().await`; non-Unix
  branch is `pending()` (correct — process termination on Windows uses
  Ctrl+C / kill which take different paths).
- `codex-rs/exec-server/src/remote.rs:194-204` — the remote-aware
  executor passes `CancellationToken::new()` (a never-cancelled token)
  to preserve current behavior. Reasonable for now but means remote
  connections can't be cleanly shut down via this mechanism.

## Concerns / nits
1. **No SIGINT handling.** `Ctrl+C` on a foreground exec-server still
   relies on the default Tokio behavior (immediate process termination),
   which means in-flight sessions don't get the same graceful drain.
   Either add `SignalKind::interrupt()` to a multi-`select!` in
   `shutdown_signal()` or document why SIGINT is intentionally
   excluded (e.g., "operators should send SIGTERM via systemd /
   launchd; SIGINT is for interactive aborts").
2. **`SessionRegistry::shutdown_all` shuts down sessions sequentially.**
   `for session in sessions { session.process.shutdown().await; }` will
   take O(N × per-session-shutdown-time) under load. For a server with
   100 detached sessions each taking 200ms to clean up, that's 20s of
   wall-clock during which new sessions can't reach the registry but
   the listener is already closed. Consider `futures::future::join_all`
   to parallelize; the existing `Mutex` is released before the loop so
   concurrent shutdowns don't deadlock against each other.
3. **Remote executor passes `CancellationToken::new()`.** This is a
   never-cancelled token, which means SIGTERM in the remote path is a
   no-op. The PR's title says "exec-server SIGTERM shutdown" so this
   is arguably out of scope, but a `// TODO: thread shutdown_token
   from caller for remote executor too` would prevent future confusion.
4. **Shutdown timeout is unbounded.** If a session's `process.shutdown()`
   hangs (e.g., child process stuck in D-state), `shutdown_all()` blocks
   forever. A `tokio::time::timeout(SHUTDOWN_TIMEOUT, ...)` wrapper around
   each shutdown call (with a `warn!` log on timeout) would bound the
   degraded-shutdown blast radius.
5. **Test in `processor.rs:347-353` only covers the never-cancelled
   token case.** A test that creates a token, spawns the connection,
   sends a `cancel()`, and asserts the connection terminates within a
   bounded `tokio::time::timeout` would lock the contract. Currently
   only the wiring is tested, not the cancellation semantics.
6. **`signal_task.abort()` on stdio happy-path exit.** Correct, but
   means if SIGTERM arrives between `connection.run()` returning and
   `signal_task.abort()` firing, the cancel is wasted (token already
   has no observers). Benign but worth a comment.

## Risk
Low-to-medium. The wiring is clean and mostly additive; the breaking
change is the `run_connection` signature which is correctly propagated to
both `remote.rs` and the test harness. The unbounded shutdown timeout and
sequential session drain are operational concerns that don't block merge
but should land before this is deployed against a heavily-trafficked
exec-server fleet.
