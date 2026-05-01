# openai/codex #20685 — Fix Windows PTY teardown by preserving ConPTY ownership

- **Repo:** openai/codex
- **PR:** https://github.com/openai/codex/pull/20685
- **HEAD SHA:** `9415d620e7bdd2ec68c4443267b2dac01e13f6ac`
- **Author:** iceweasel-oai
- **Verdict:** `merge-after-nits`

## What the diff does

Closes a Windows-only "background terminal stays visible after the
shell process exits" bug rooted in a ConPTY teardown ordering
violation. The elevated runner waits for the PTY output reader to
reach EOF before sending the final `ExitPayload`, but the prior
ConPTY helper reduced the rich `PsuedoCon` owner type down to a raw
`HANDLE` triple (`hpc`, `input_write`, `output_read`) immediately
after creation. That detached the OS-level `PSEUDOCONSOLE` lifetime
from the handle wrapper, which then meant `ClosePseudoConsole` ran
*after* the calling code had already moved on, leaving the
backing pipe handles (`input_write` / `output_read`) alive past
teardown. EOF on the output pipe never propagated, the output
reader never returned, and the runner's "wait for reader EOF before
emitting Exit" gate never fired — session stayed `running` forever
in the UI.

Fix: keep the typed `PsuedoCon` owner alive through the entire
spawn-to-teardown lifecycle, hand out only the borrowed pipe
handles the runner actually needs, and drop the owner *last* so
`ClosePseudoConsole` runs after every borrowed handle is already
released.

Files of note:

- `utils/pty/src/win/conpty.rs:82-93` — `into_raw_handles` →
  `into_handles` rename, returning `(PsuedoCon, FileDescriptor,
  FileDescriptor)` (typed-owner triple) instead of `(RawHandle,
  RawHandle, RawHandle)` (raw-handle triple). Implementation
  uses `ManuallyDrop` + three `ptr::read` calls to byte-move the
  fields out of `self` without invoking `Drop` on the original
  struct (the `ManuallyDrop` ensures the source `RawConPty`'s
  destructor doesn't double-close the handles after they've been
  moved out). This is the load-bearing primitive — without it,
  the source struct's `Drop` would close the same handles the
  caller now owns.
- `utils/pty/src/lib.rs:34-39` and `utils/pty/src/win/mod.rs:51` —
  re-export `PsuedoCon` so downstream crates can hold the typed
  owner.
- `windows-sandbox-rs/src/conpty/mod.rs:34-66` — `ConptyInstance`
  is restructured: the public `pub hpc: HANDLE` /
  `pub input_write: HANDLE` / `pub output_read: HANDLE` /
  `desktop: Option<LaunchDesktop>` field set is replaced with a
  private `pseudoconsole: Option<PsuedoCon>` plus the same two
  pipe handles plus a renamed `_desktop` (leading-underscore to
  silence the unused warning while keeping the field around for
  Drop ordering). Drop impl at `:46-59` closes the two pipe
  handles first (`CloseHandle` on each if non-null and
  non-`INVALID_HANDLE_VALUE`), then `let _ = self.pseudoconsole.
  take();` *after* both handles are closed — this is the right
  order because `ClosePseudoConsole` blocks until all references
  drain and would otherwise wedge if the pipe handles are still
  alive. The accessors at `:64-75` (`raw_handle()`,
  `take_input_write()`, `take_output_read()`) replace the prior
  `into_raw(self) -> (HANDLE, HANDLE, HANDLE,
  Option<LaunchDesktop>)` API — callers now take only the
  specific handles they need, and the `ConptyInstance` owner
  stays alive for the duration of the session.
- `windows-sandbox-rs/src/conpty/mod.rs:78-130` — the two creation
  sites (`create_conpty` and `spawn_conpty_process_as_user`) are
  updated to consume the new `into_handles` triple and call
  `into_raw_handle()` on the `FileDescriptor` values to get the
  `HANDLE` representations they need to store in `ConptyInstance`.
  `IntoRawHandle` (rather than `AsRawHandle`) is the correct trait
  here because we're *transferring* ownership of the handle out
  of the `FileDescriptor` wrapper and into the
  `ConptyInstance.input_write` / `output_read` fields, so the
  `FileDescriptor` should *not* close the handle when it drops.
- `windows-sandbox-rs/src/elevated/command_runner_win.rs:85-91,
  263-279,525-611` — replaces the prior `into_raw(self)
  destructure-everything-immediately` pattern with `take_*` on the
  borrowed handles followed by holding `conpty_owner: Some(conpty)`
  in the `IpcSpawnedProcess` struct. The teardown path at `:608-611`
  used to do `unsafe { ClosePseudoConsole(hpc); }` directly; now
  it does `let _ = guard.take();` (clears the optional handle from
  the shared mutex without calling `ClosePseudoConsole`, which is
  now handled by `ConptyInstance`'s Drop) and `drop(conpty_owner.
  take());` *after* the output reader thread has joined. The
  removal of the `use ClosePseudoConsole` import at `:60` is the
  signal that no caller is doing manual close anymore — the
  responsibility has moved into `ConptyInstance::Drop`, which is
  the right place for it.
- `windows-sandbox-rs/src/unified_exec/backends/legacy.rs:48-130,
  331-396` — same restructuring applied to the legacy unified-exec
  backend: `LegacyProcessHandles` gains a `conpty_owner:
  Option<ConptyInstance>` field, the `spawn_legacy_process` arm
  for `tty=true` does `conpty.take_output_read()` and
  `conpty.take_input_write()` before passing the rest of the
  `ConptyInstance` into `LegacyProcessHandles`, and the teardown
  does `drop(conpty_owner.take());` after every reader/writer
  handle is joined.

## Why it's right

The diagnosis nails the actual failure mode: ConPTY teardown is
ordering-sensitive (`ClosePseudoConsole` blocks waiting for
references to drain, but if those references are detached raw
handles the OS doesn't know they belong to that pseudoconsole), and
the prior `into_raw_handles()` API made the ordering invariant
impossible to express in the type system because all three handles
came back as bare `HANDLE`/`RawHandle` values with no Drop
relationship.

The restructured `ConptyInstance::Drop` at `conpty/mod.rs:46-59`
puts the order explicitly: pipe handles first (`CloseHandle` on
both), then `pseudoconsole.take()` (which invokes `PsuedoCon::Drop`,
which calls `ClosePseudoConsole` internally). That order is
load-bearing because `ClosePseudoConsole` will not return until the
backing pipe references drain — closing the pipes first ensures the
drain completes promptly. The prior code closed the pseudoconsole
handle inside `ConptyInstance::Drop` too but was working with raw
detached `HANDLE` values that had no relationship to the
`PsuedoCon` wrapper that was already dropped, so the drain never
finished promptly and downstream readers blocked forever waiting
for EOF.

Switching from `AsRawHandle` to `IntoRawHandle` at
`conpty/mod.rs:122-123` is the right call: the
`FileDescriptor::into_raw_handle()` consumes `self` and returns
the underlying `HANDLE` *without* invoking `Drop`, which is exactly
what's needed when transferring handle ownership from the
`FileDescriptor` wrapper into the `ConptyInstance` struct that will
own it from then on. Using `as_raw_handle()` instead would have
double-freed the handle (once by the `FileDescriptor`'s Drop, once
by `ConptyInstance::Drop`'s `CloseHandle`).

The two teardown call sites
(`command_runner_win.rs:608-611` and `legacy.rs:392-395`) both
follow the same pattern: `let _ = guard.take();` to drop the
optional `HANDLE` reference without doing a manual close, then
`drop(conpty_owner.take());` to invoke the `ConptyInstance::Drop`
that does the correct ordering. Reader thread joined → pipe handles
closed inside Drop → `ClosePseudoConsole` runs last → no orphaned
session in the UI.

The validation in the PR body (manual desktop-app testing with
11×`ping -n 3` and 5×`ping -n 30` cleanly draining) is the right
shape for a teardown-ordering bug — there's no clean unit-test
surface for "the pseudoconsole gets closed in the right order
relative to its pipe handles," and a manual stress run that
matches the original symptom report (sessions accumulating in the
UI) is the canonical regression check.

## Nits

1. **No automated regression test.** Understandable — this is a
   Windows-OS-level lifecycle ordering bug that's hard to assert
   in unit tests — but a Windows-CI integration test that spawns
   `cmd /c "ping -n 1 127.0.0.1"` 5× in sequence and asserts that
   each session's `ExitPayload` arrives within (e.g.) 10 seconds
   would catch a future regression that breaks the ordering
   invariant. Without that, the next refactor that touches the
   `Drop` order risks reintroducing the symptom and only catching
   it in manual QA.

2. **`PsuedoCon` typo in the public re-export** (`utils/pty/src/
   win/mod.rs:51`) — the prior code already had this typo
   (should be `PseudoCon`) but this PR makes it a public re-export
   from `codex_utils_pty`, baking the typo into a wider API
   surface. If the rename to `PseudoCon` is feasible without
   breaking external callers, this is the moment to do it; if
   not, a `#[allow(non_camel_case_types)]`-ish doc comment
   acknowledging the typo would help the next reader.

3. **`Drop` order at `conpty/mod.rs:46-59`** — `unsafe { if
   self.input_write != 0 && self.input_write != INVALID_HANDLE_VALUE
   { CloseHandle(self.input_write); } if self.output_read != 0 &&
   self.output_read != INVALID_HANDLE_VALUE { CloseHandle(self.
   output_read); } }` then `let _ = self.pseudoconsole.take();`
   *after* the unsafe block. The `take()` is at safe-Rust scope,
   which is fine, but a comment explaining "close pipes first,
   then drop the pseudoconsole owner so `ClosePseudoConsole` runs
   after the drain completes" would lock the order against future
   "let me clean this up" refactors that re-order on aesthetics.

4. **`take_input_write` / `take_output_read` at `:71-75`** use
   `std::mem::replace(&mut self.input_write, 0)` rather than
   `Option::take()`. `0` and `INVALID_HANDLE_VALUE` are both
   sentinel "no handle" values for `HANDLE`, and the Drop check
   handles both — but representing the post-take state with a
   typed `Option<HANDLE>` would make "this handle has been
   transferred out" express in the type system rather than via
   sentinel values. Mechanical refactor; the current form works.

5. **`IpcSpawnedProcess.conpty_owner` and `LegacyProcessHandles.
   conpty_owner` are sibling-named to but separate from the prior
   `_desktop_owner` / `desktop` fields** — fine for now, but if a
   future change consolidates "things that must outlive the
   spawned process" into a single owner struct, both these
   refactors should be done together.

6. **The `command_runner_win.rs:608-611` teardown** does `let _ =
   guard.take();` before `drop(conpty_owner.take());`. Both lines
   are needed (the `guard.take()` clears the `Arc<Mutex<Option<
   HANDLE>>>` that was used elsewhere in the file to track the
   raw `hpc_handle` for resize support), but a one-line comment
   explaining why we need both — that the `hpc_handle` mutex is
   for `ResizePseudoConsole` callers and is conceptually a
   *reference* to the same resource owned by `conpty_owner` — would
   prevent a future "looks redundant, let me delete one" cleanup.

## Verdict rationale

Right diagnosis (ConPTY teardown ordering, raw-handle detachment
breaking the drain), right primitive (`ManuallyDrop` + `ptr::read`
to byte-move owned fields out of the source struct without
invoking its Drop), right load-bearing change (typed `PsuedoCon`
owner held alive through the session lifecycle, dropped *after*
the pipe handles are closed inside `ConptyInstance::Drop`), right
trait choice (`IntoRawHandle` for the ownership transfer from
`FileDescriptor` into `ConptyInstance`), validation matches the
original symptom shape (sessions cleanly draining instead of
accumulating). Wants automated Windows-CI integration test for
session-drain timing, sentinel-`HANDLE` → `Option<HANDLE>` cleanup,
and a doc comment on the load-bearing Drop order before a future
aesthetic refactor accidentally re-orders.

`merge-after-nits`
