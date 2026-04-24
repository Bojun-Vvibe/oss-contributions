# codex PR #19211 — [codex] Fix Windows process management edge cases

Link: https://github.com/openai/codex/pull/19211

## What it changes

Follow-up to a larger Windows session/elevation PR. Concrete fixes:

- bound the elevated-runner pipe-connect handshake (no more infinite
  block on a stuck named pipe);
- if the bounded handshake fails, terminate the spawned runner so we
  do not leak `codex-command-runner.exe` processes;
- loop on partial `WriteFile` returns when forwarding stdin into the
  elevated runner, fixing silent stdin truncation;
- tighten HANDLE / SID cleanup in the runner setup path;
- keep draining driver-backed stdout/stderr until the backend closes
  rather than dropping the tail at a fixed 200ms grace;
- introduce a `LocalSid` RAII wrapper for SID ownership.

~338 added / 69 deleted across `utils/pty` and `windows-sandbox-rs`.
Validation: `cargo fmt`, scoped `cargo test` for both crates,
explicitly notes the broader crate test suite has unrelated
PowerShell-dependent failures.

## Critique

Each individual fix is recognizably correct and the scope discipline
is good — the PR explicitly defers the larger helper-thread
ownership refactor. Three things worth pushing on:

1. **Bounded handshake timeout value.** A bounded wait on the pipe
   connect is the right shape, but the timeout magnitude matters:
   too short and slow elevation prompts (UAC waiting for user
   click) get a spurious failure; too long and a stuck runner hangs
   the whole session. The PR should expose the timeout as a const
   at the top of `runner_client.rs` with a comment justifying the
   value, so a future tuner does not have to spelunk.
2. **Tail-drain termination condition.** "Drain until the backend
   closes" is correct in principle, but on Windows
   `ReadFile`-on-pipe can hang if the backend dies without closing
   its write end (process killed by AV, OOM-killed, etc.). The
   drain loop needs a *secondary* stop condition — e.g., the child
   process handle being signaled — or the previous 200ms grace
   period reappears as a different bug (drain hangs forever).
3. **`WriteFile` partial-write loop.** Correct fix. Worth asserting
   `bytes_written > 0` per iteration and bailing on a zero-progress
   write to avoid a busy loop if the pipe enters a degenerate
   state.
4. **Test coverage for the leak.** `runner_client.rs` gains 159
   lines; the new tests cover handshake-timeout / handshake-error
   paths, but the *process-leak* claim ("orphaned runner on
   timeout") is the most user-visible bug and deserves an explicit
   "after timeout, no `codex-command-runner.exe` survives" assert
   even if it has to be gated to Windows CI.

## Suggestions

- Pull the handshake timeout to a named const with justification.
- Add a child-process-signaled escape hatch to the tail-drain loop.
- Add an explicit no-orphan-process assertion to the timeout test
  path.

Verdict: solid follow-up. Land after timeout-naming and tail-drain
escape hatch are addressed.
