# charmbracelet/crush PR #2606 — feat: split-pane tree, tab manager, and cross-platform PTY

- **Author:** ahostbr
- **Head SHA:** 972b135b66f46f9ac99a824d8d64de3cc5ee5cf7
- **Files:** 15 new files, +3085 / −0 (no integration with existing
  TUI yet)
- **Verdict:** `request-changes`

## What the diff does

Adds three brand-new packages — none of which is wired into the
running app:

- `internal/pty/` (+474): cross-platform PTY abstraction.
  `pty.go` defines the `PTY` struct + `Options`, `pty_unix.go`
  (+58) is the planned creack/pty backend, `pty_windows.go` (+228)
  uses ConPTY via direct `CreateProcessW` syscalls.
- `internal/split/` (+1169): split-pane tree with separate `tree.go`
  (+219), `layout.go` (+180), `dirty.go` (+131) plus tests
  (`tree_test.go` +265, `layout_test.go` +242, `dirty_test.go` +132).
- `internal/tabmgr/` (+1442): tab manager with `tab.go` (+244),
  `tabmgr.go` (+224), `persistence.go` (+247) and tests
  (`tab_layout_test.go` +193, `tabmgr_test.go` +317,
  `persistence_test.go` +217).

## Review notes

- **Nothing imports any of this.** Grep across `internal/ui`,
  `internal/agent`, `cmd/` etc. shows no consumers of `internal/pty`,
  `internal/split`, or `internal/tabmgr`. Adding 3000 lines of
  *unused* infrastructure is a maintainability hazard: it grows the
  test surface, the build time, and the API freeze radius without
  delivering user value, and the next refactor will touch it
  blind.
- **PR title oversells.** "split-pane tree, tab manager, and
  cross-platform PTY" sounds like a feature; it's actually three
  building blocks for a future feature. The PR body should make
  clear what user-visible thing this *enables*, and ideally include
  the consumer in the same PR (or at least a draft follow-up).
- **`pty/pty.go` line 22 stores ConPTY handles as `interface{}`**
  with the comment "Platform-specific handle (ConPTY HPCON on
  Windows, *os.File on Unix)." This is a smell — the cross-platform
  abstraction can hide the type behind build tags
  (`pty_unix.go` and `pty_windows.go` already exist as separate
  files). A `type ptyHandle = ...` per-OS alias removes the
  `interface{}` and the runtime type assertions it implies.
- **`pty.go` line 25**: `procHandle uintptr` "stored as uintptr for
  cross-platform compilation" — same issue. Use platform-specific
  fields in the OS files; the cross-platform `PTY` struct should
  be opaque about handles.
- **`pty_unix.go` is 58 lines and the comment in `pty.go` line 4
  says "uses POSIX openpty via creack/pty (planned)"** — i.e. the
  Unix backend isn't actually finished. Shipping a "cross-platform
  PTY" where one platform is a stub is a bigger problem than the
  unused-code issue above. Either finish Unix or scope the PR to
  Windows only and rename it.
- **Direct `CreateProcessW` syscall on Windows** instead of using
  `os/exec` + ConPTY libraries (e.g. UTF-8 console, `xtermjs/winpty`
  bindings, or the well-trodden `creack/pty` Windows path) is a
  maintenance burden. There should be a strong reason in the PR
  description; absent that, prefer the established library route.
- **DefaultShell on Windows** prefers `pwsh.exe` then falls back to
  `powershell.exe` (line 56–60). Reasonable. But there's no
  `COMSPEC` / `cmd.exe` ultimate fallback — on stripped-down
  Windows containers neither PowerShell may exist.
- **Tests are extensive** (±1300 lines of test code for the new
  packages) which is good — but they all test the new packages in
  isolation. There's no integration test exercising `tabmgr`
  driving `split` driving `pty`, which is the entire point of
  shipping the three together.

Recommend: split into (1) the tabmgr + split packages with at least
one stub consumer in `internal/ui`, (2) PTY package landed only when
the Unix backend exists, (3) actual UI integration as the PR that
demonstrates the design works end-to-end.

## What I learned

"Cross-platform" abstractions that use `interface{}` for the
platform-specific bits are usually a sign that build tags weren't
used aggressively enough; pushing the type difference to the file
boundary (per-OS files in the same package) keeps the public API
typed and the unsafe bits localized.
