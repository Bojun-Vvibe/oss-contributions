# block/goose PR #9023 — fix(acp): synchronously reap ACP child to avoid SIGCHLD race

- URL: https://github.com/block/goose/pull/9023
- Head SHA: `fa2cc085d1b8faefacf9d6abaf6410d2769d48a5`
- Author: (supersedes #8790, fixes #8373)
- Size: +23 / -2 (one file: `crates/goose/src/acp/provider.rs`)

## Verdict
`merge-as-is`

## Rationale

This is a precise diagnosis and minimal fix for a real correctness bug. The `console` crate's use
of `select()` treats *any* interruption as a fatal error and propagates it; the previous code at
`provider.rs:651` returned from `run_with_child(...)` without first waiting for the ACP child to be
reaped, so the kernel could deliver SIGCHLD interrupting an in-flight `select()` in `console`,
which `console` then re-raised as fatal — causing the goose process to exit instead of cleanly
shutting down the ACP session. The fix at `provider.rs:651-657` captures the run result, then
explicitly `child.kill().await` and `child.wait().await` *before* returning, so the SIGCHLD has
already been consumed by `wait()` and there's no remaining unreaped child to interrupt the next
`select()`. Both calls correctly use `let _ =` discard because at this point the child's exit code
is irrelevant — the higher-level transport already either succeeded or returned the real error.

The orthogonal stderr change is the right pairing. Switching `spawn_acp_process` at `provider.rs:898`
from `Stdio::inherit()` to `Stdio::piped()` and then forwarding child stderr through
`forward_child_stderr()` at `provider.rs:882-893` (line-buffered via `BufReader::new(stderr).lines()`,
emitted at `tracing::info!(target: "acp::child::stderr", ...)`) prevents the previously-reported
class of bugs where ACP children that write to stderr scribbled directly onto the user's CLI and
broke the TUI's render. The early `child.stderr.take()` at `:651-653` is correct: it has to happen
*before* the `transport` is constructed and `self.run(...)` is awaited, otherwise the spawned
forwarder races against the child's first stderr line. `tokio::spawn` here is the right choice
because the forwarder needs to keep draining stderr for the child's lifetime independent of the
transport's read/write loop, and the forwarder's loop terminates cleanly on `Ok(None)` (EOF on
child exit) which is exactly when `child.wait().await` returns at `:656`.

Test/coverage note for the merger but not blocking: there's no new automated test in this PR
because the bug only reproduces against the real `console` crate's `select()` interaction with
SIGCHLD, which is a kernel-timing race that's hard to make deterministic in unit tests. The fix is
small enough and the diagnosis (linked-issue + superseded PR) is well-documented enough that
landing without a regression test is the right call here — adding a fragile flake-prone test would
be net negative. Ready to land.
