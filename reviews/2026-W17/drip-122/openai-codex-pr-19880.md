---
pr: 19880
repo: openai/codex
sha: 6502bbed960f2cfc3b09857426743d607bbc0d39
verdict: merge-after-nits
date: 2026-04-28
---

# openai/codex #19880 — fix: cancel Windows sandbox on network denial

- **Author**: viyatb-oai
- **Head SHA**: `6502bbed960f2cfc3b09857426743d607bbc0d39`
- **Size**: 224 added / 17 deleted across 4 files (`core/src/exec.rs`,
  `windows-sandbox-rs/src/{elevated_impl.rs,lib.rs,unified_exec/tests.rs}`).
- **Stacked on**: `codex/viyatb/fix-network-proxy-denials`.

## Scope

Closes the gap where a Guardian/proxy network denial inside the Windows
sandbox capture path could only terminate the child process by waiting for
the timeout to elapse, even when the caller had already cancelled via
`ExecExpiration::Cancellation` / `TimeoutOrCancellation`. Adds a
`WindowsSandboxCancellationToken` predicate primitive at the crate boundary,
threads it through `ExecExpiration::cancellation_token()` → both elevated and
non-elevated capture entry points, and spawns a small parker-thread that
writes a `Message::Terminate` framed message to the runner pipe as soon as
the cancellation predicate flips.

## Specific findings

- `codex-rs/windows-sandbox-rs/src/lib.rs:11-35` — `WindowsSandboxCancellationToken`
  wraps `Arc<dyn Fn() -> bool + Send + Sync>`. Reasonable shape — keeps the
  `windows-sandbox-rs` crate independent of `tokio_util::sync::CancellationToken`
  (which lives in `core`), and the closure-based contract is the right
  abstraction for "is this still alive?". The `Debug` impl is
  `finish_non_exhaustive()` — correct since the closure is not introspectable.
- `codex-rs/core/src/exec.rs:225-233` — new
  `ExecExpiration::cancellation_token()` matches `Cancellation` and
  `TimeoutOrCancellation` arms, returns `None` for the two timeout-only
  arms. Correct, and the `#[cfg_attr(not(target_os = "windows"), allow(dead_code))]`
  is the right knob for the cross-platform crate to compile cleanly.
- `codex-rs/core/src/exec.rs:601-614` — the construction site. The earlier
  TODO `// TODO(iceweasel-oai): run_windows_sandbox_capture should support all
  variants of ExecExpiration, not just timeout.` is correctly removed and
  replaced with the new `cancellation` plumbing. Note the timeout-and-cancel
  are still passed as **two separate arguments**, not a single
  `ExecExpiration` value — that's a deliberate scope-limit (avoids touching
  the IPC argument shape) but means a future refactor will eventually want
  to collapse them.
- `codex-rs/windows-sandbox-rs/src/elevated_impl.rs:118-145` —
  `spawn_cancel_writer(pipe_write, cancellation)` is the load-bearing piece.
  Spawns an OS thread (not async — `pipe_write` is a `std::fs::File` and
  the framing API is sync), polls `cancellation.is_cancelled()` every 50ms
  via `thread::park_timeout(Duration::from_millis(50))`, and on detection
  writes a `FramedMessage { version: 1, message: Message::Terminate { .. } }`
  to the pipe. The `Arc<AtomicBool>` `done` flag is the shutdown signal so
  the thread exits cleanly when the main loop completes normally.
- `codex-rs/windows-sandbox-rs/src/elevated_impl.rs:246-285` — the main
  capture loop is rewritten from `?`-propagating `read_frame(...)?
  .ok_or_else(...)?` to a `match read_frame(...)` whose error/None arms
  break with `Err(...)` rather than returning, and the success arms break
  with `Ok((exit_code, timed_out))`. Necessary so the cancel-writer
  cleanup at `:286-291` (`done.store(true); cancel_handle.thread().unpark();
  let _ = cancel_handle.join(); drop(pipe_write); let (exit, timed) =
  result?;`) always runs even on read error — without this restructure,
  a `?` short-circuit would have leaked the cancel thread.
- `codex-rs/windows-sandbox-rs/src/elevated_impl.rs:264-272` — `decode_bytes`
  errors are now also caught by the loop (was: propagated). Same reasoning
  as above — keeps the cancel-writer cleanup reachable on every exit path.
- `codex-rs/windows-sandbox-rs/src/unified_exec/tests.rs:+52` — adds a
  regression test that cancellation is not reported as timeout. This is the
  exact failure mode to pin: without the new wiring, the runner would have
  exited via timeout and `timed_out: true` would propagate up, masking the
  cancel cause. With the wiring, the `Terminate` message takes effect first
  and `timed_out: false` is correctly returned.

### Real nits

- The 50ms `park_timeout` poll interval is a magic number with no constant
  name and no rationale. Cancellation latency budget here is 0–50ms on top
  of however long the Guardian denial took to detect — that's fine for the
  proxy-denial use case but worth naming as e.g.
  `const CANCEL_POLL_INTERVAL: Duration = Duration::from_millis(50)` with
  a one-line comment explaining the budget.
- `cancel_handle.thread().unpark()` is called *before* `cancel_handle.join()`
  to wake the parker so it observes `done` immediately. That's correct, but
  someone reading this in 6 months will wonder why both calls are needed. A
  one-line comment (`// unpark wakes the polling thread immediately so we
  don't wait the full park_timeout window during shutdown`) would close the
  gap.
- The `try_clone()?` at `:122` on `pipe_write` (so the cancel writer and
  the main loop both own a handle) is correct, but the error path leaks the
  cancel writer thread if `try_clone` fails after the predicate path returns
  — actually no, `try_clone` is the first line so it returns before the
  thread is spawned. Fine.
- `exec.rs:601-602` — comment changed from `// TODO ... should support all
  variants of ExecExpiration, not just timeout.` to `// Windows sandbox
  capture still receives timeout and cancellation separately.` Technically
  accurate but loses the "and we should clean this up someday" signal —
  worth keeping an explicit `// FOLLOWUP: collapse into one ExecExpiration
  argument` or similar.

## Risk

Medium. Real cross-process cleanup logic with a polling thread, IPC
framing, and an atomic flag — meaningful surface for races. The most
plausible races (cancel during pipe-close, double-shutdown, missed wakeup)
all appear handled: `done.store(true)` before `unpark()` ensures the
thread observes the flag on its next check; `let _ = cancel_handle.join()`
absorbs panics; `drop(pipe_write)` after the join ensures the runner
sees EOF only after the cancel attempt completes.

The cancel path is fire-and-forget on the `write_frame(...)` result
(`let _ = ...`). That's the right call — if the runner has already
exited, the write fails harmlessly; if the pipe is broken, the main
loop will see the error and break with `Err(...)`. But this means there's
no observable signal anywhere if the cancel write fails for an unexpected
reason. A `tracing::debug!` on the write error would help diagnose
future "cancel got swallowed" reports.

## Verdict

**merge-after-nits** — name the 50ms polling constant, add the
unpark-then-join comment, restore an explicit follow-up marker for the
"two separate args" shape, and add a `tracing::debug!` on the
`write_frame` failure inside the cancel writer. The architecture is
correct.

## What I learned

The "rewrite the loop from `?`-propagation to `match-and-break-with-result`"
pattern when adding a teardown step is one of those refactors that looks
mechanical but has real correctness content. Every `?` in the original
loop body would have skipped the cancel-writer cleanup; the audit you
have to do to convince yourself the new shape is right is exactly the
kind of audit that catches dropped cleanups in similar code elsewhere.
Worth standardizing this transformation for any future "I need to clean
up an OS resource on every exit path" change.
