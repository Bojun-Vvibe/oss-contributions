# openai/codex PR #20685 — Fix Windows PTY teardown by preserving ConPTY ownership

- PR: https://github.com/openai/codex/pull/20685
- Head SHA: `9415d620e7bdd2ec68c4443267b2dac01e13f6ac`
- Author: @iceweasel-oai
- Size: +70 / -57

## Summary

On Windows, the elevated runner waits for the PTY's output reader to hit EOF before emitting the final exit message to the desktop UI. The previous `RawConPty::into_raw_handles` returned bare `RawHandle`s and dropped the owning `PsuedoCon` / `FileDescriptor` wrappers, which meant the borrowed pipe handles were closed prematurely — but the *pseudoconsole* itself was kept alive separately and closed manually with `ClosePseudoConsole` at teardown. The combination produced a window where the pipes were closed but the pseudoconsole hadn't been, EOF never propagated to the reader, and the UI showed a "running" terminal forever.

This PR replaces `into_raw_handles` with `into_handles` returning the owned `(PsuedoCon, FileDescriptor, FileDescriptor)` tuple. Callers (`windows-sandbox-rs`, the elevated command runner, the legacy unified-exec backend) now hold the `PsuedoCon` owner alongside the pipe handles for the lifetime of the session and drop them together; the manual `ClosePseudoConsole` calls are removed.

## Specific references from the diff

- `codex-rs/utils/pty/src/win/conpty.rs:85-93` — `into_raw_handles` → `into_handles`. Uses `ManuallyDrop` + `ptr::read` to move the three owned fields out of `self` without running `Drop`. Functionally fine but is `unsafe`; the safety argument (no further use of `me`, all three fields are independently owned) is implicit.
- `codex-rs/utils/pty/src/lib.rs:36-38` — re-exports `PsuedoCon` so downstream crates can name the owner type. Necessary for the new public API.
- `codex-rs/windows-sandbox-rs/src/conpty/mod.rs:34-58` — `ConptyInstance` now holds `Option<PsuedoCon>` instead of a raw `HANDLE hpc`. `Drop` lets the `PsuedoCon` drop naturally (which in turn calls `ClosePseudoConsole`), eliminating the manual call.
- `codex-rs/windows-sandbox-rs/src/conpty/mod.rs:60-72` — replaces `into_raw` with three accessors: `raw_handle()` (borrows), `take_input_write()`, `take_output_read()` (move-out via `mem::replace`). Cleaner ownership story than the prior all-or-nothing `into_raw`.
- `codex-rs/windows-sandbox-rs/src/elevated/command_runner_win.rs:88` — `IpcSpawnedProcess` adds `conpty_owner: Option<ConptyInstance>` and removes `_desktop_owner: Option<LaunchDesktop>`. The `LaunchDesktop` is now transitively owned through `ConptyInstance._desktop`, which is the right place for it. (Imports for `LaunchDesktop` and `ClosePseudoConsole` are correctly removed at lines 60 and 88 of the original imports block.)

## Verdict: `merge-after-nits`

The diagnosis is precise (pipe-handle EOF starvation on early ownership drop), and the fix moves the codebase from a fragile "raw handles + side-channel close" model to a single-owner RAII model. The validation evidence — 11 short pings + 5 long pings draining cleanly from the desktop UI — directly exercises the bug. Two real concerns keep this off `merge-as-is`.

## Nits / concerns

1. **`into_handles` is `unsafe { ptr::read(...) }` × 3 with no `// SAFETY:` comment.** At `conpty.rs:88-92` the three `ptr::read` calls duplicate ownership of `con`, `input_write`, `output_read` out of `me: ManuallyDrop<Self>`. This is sound *because* `me` is wrapped in `ManuallyDrop` and never re-read or dropped — but the next person to refactor this method will not see why it's safe. Add a `// SAFETY:` comment explicitly noting (a) `ManuallyDrop` suppresses `Self::drop`, (b) each field is independently owned and not aliased, (c) `me` is shadowed/dead after the tuple is constructed.
2. **`Drop` order in `ConptyInstance` matters.** At `windows-sandbox-rs/src/conpty/mod.rs:42-53`, `Drop::drop` first calls `CloseHandle(self.input_write)` and `CloseHandle(self.output_read)`, then `let _ = self.pseudoconsole.take();` (which drops the `PsuedoCon`, which calls `ClosePseudoConsole`). This is the *opposite* of the order the original bug suggested was needed — the bug was that pipes closed before the pseudoconsole. Closing pipes first then the pseudoconsole is correct for *this* drop path (we're tearing down at session end), but please double-check that no caller is observing EOF in another thread *between* the two `CloseHandle` calls and the `pseudoconsole.take()` line. If yes, swap the order or move the pseudoconsole drop before the pipe closes.
3. **`ConptyInstance` no longer exposes `hpc` as a struct field**, but `spawn_conpty_process_as_user` at lines 117-127 reads the handle once into a local `hpc` *before* moving the `pseudoconsole` into the struct. Cleaner: construct the struct first, then call `conpty.raw_handle().expect(...)` for the `attrs.set_pseudoconsole(...)` call. As written it's correct, just oddly ordered.
4. **No new test for the EOF-propagation path.** Validation is manual (UI smoke tests). An integration test in `windows-sandbox-rs` that creates a `ConptyInstance`, spawns `cmd /c exit 0`, and asserts the output reader hits EOF within N ms would lock the regression. I understand Windows-only PTY tests are painful in CI, but the validation table in the description suggests this is reachable from the existing test harness.
5. **`take_input_write` / `take_output_read` leave a `0` HANDLE behind** (via `mem::replace(&mut self.input_write, 0)`). The `Drop` impl at lines 44-50 then has `if self.input_write != 0 && self.input_write != INVALID_HANDLE_VALUE` guards which correctly skip closing. Good defensive code; worth a one-line comment that the `0` sentinel is intentional and `Drop` is aware of it.
