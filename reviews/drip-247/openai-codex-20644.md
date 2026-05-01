# openai/codex #20644 — Fix elevated Windows PTY teardown after shell exit

- Link: https://github.com/openai/codex/pull/20644
- Head SHA: `53e648595898f94ef3afdba828e76bacb5b9ae05`
- Author: iceweasel-oai

## Summary
Background terminals on Windows outlived their shell because the elevated command runner kept the ConPTY input-pipe HANDLE alive in the input thread, which let `conhost.exe` hold the pseudoconsole open even after the child shell exited. Net effect: the output reader never reached EOF, the runner never sent `Message::Exit`, and from unified-exec's view the PTY session never ended — the background-terminals UI grew without bound. Fix promotes the stdin HANDLE from a thread-local `Option<HANDLE>` to a shared `Arc<StdMutex<Option<HANDLE>>>` so the main teardown path can `take()` and `CloseHandle` it as soon as the child shell exits, *before* closing the pseudoconsole and waiting for the output reader to drain.

## Line-level observations
- `windows-sandbox-rs/src/elevated/command_runner_win.rs:363-368` — the lifetime promotion is the structural fix. The lengthy comment at `:363-367` explaining "either side can `take()` ownership exactly once" is operator-grade — a future reader will understand *why* the mutex exists rather than refactoring it back to a thread-local.
- `:384-391` — every read inside `Message::Stdin` arm now grabs `stdin_handle.lock()` then `.as_ref().copied()` to clone the HANDLE value out for the per-message write. The `if let Ok(mut guard) = stdin_handle.lock() && let Some(handle) = guard.as_ref().copied()` pair is correct: HANDLE is `Copy` (it's a `*mut c_void`-shaped winapi type) so `.copied()` is the right way to get the value without taking ownership during a write.
- `:418-435` — the partial-write retry loop preserves the prior fail-path semantics (on `WriteFile` failure or zero-byte write, close the handle and clear it) but now writes through `*guard = None` instead of `stdin_handle = None`, which correctly publishes the cleared state to the main thread via the mutex. The lock is held for the duration of one write, which serializes against the main-thread `take()` — required for correctness.
- `:441-446` — `Message::CloseStdin` now uses `guard.take()` instead of `stdin_handle.take()`. Same shape, same semantics, lock-mediated.
- `:481-486` — the input-loop tail cleanup arm (when the read loop terminates because the IPC pipe closed) also flips to `guard.take()`. Correct: if the main thread already closed stdin from below, this arm sees `guard = None` and does nothing, idempotent.
- `:540-545` — main-line documentation of the HANDLE promotion is excellent: explicitly cites `conhost.exe` keeping the pseudoconsole alive, the failure mode (output reader never EOFs, no `Message::Exit`), and the architectural fix. This is the single comment that makes the whole change reviewable.
- `:625-635` — the new teardown gate at `req.tty && stdin_handle.lock().take()` is the load-bearing close path. Placed after the child-process-exit signal but *before* `hpc_handle.lock().take()` (the `ClosePseudoConsole` call further down), which is exactly the documented ordering required for ConPTY teardown reliability. The `req.tty` guard is correct: non-PTY callers don't have a ConPTY to unwind so the close would be redundant.
- The `Arc<StdMutex<...>>` adds one allocation per spawn but contention is naturally low (input thread holds the lock for one write at a time; main thread takes it once at teardown). No risk of priority inversion since both sides do bounded work under the lock.
- Verification path is empirical (desktop harness shows short-lived sessions now reach normal exit, longer-lived sessions disappear after they finish). No automated regression test added — Windows PTY teardown is genuinely hard to unit-test without a fake `conhost`, so this is defensible, but a comment near `:625` saying "if you change this ordering, re-run the desktop harness PTY-shrinkage test" would lock the fix's intent.

## Verdict
`merge-after-nits` — add a one-line comment near the new teardown gate at `:625` flagging the ordering invariant against future refactors, and consider whether the same shared-handle pattern applies to `stdout_handle` (probably not, since stdout is read-only by the input thread, but worth confirming). Fix is structurally correct.
