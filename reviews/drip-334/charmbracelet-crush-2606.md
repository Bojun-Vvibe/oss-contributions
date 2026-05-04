# charmbracelet/crush#2606 — feat: split-pane tree, tab manager, cross-platform PTY

- **Head SHA:** `972b135b66f46f9ac99a824d8d64de3cc5ee5cf7`
- **Author:** community
- **Size:** +3085 / −0, 15 files, 3 net-new packages

## Summary

Three foundational packages added under `internal/`: `split/` (binary split tree for pane layout), `tabmgr/` (tab manager with persistence), `pty/` (cross-platform PTY using `creack/pty` on Unix and direct ConPTY syscalls on Windows). No existing code is modified — this is pure infrastructure for future terminal multiplexing UI.

## Specific citations

- `internal/pty/pty.go:14-32` — `PTY` struct holds `cmd *exec.Cmd` (Unix path), `procHandle uintptr` (Windows path), and a generic `handle interface{}` for the platform-specific OS resource (`*os.File` on Unix, ConPTY HPCON on Windows). Uses `sync.Mutex` for `Resize`/`Close`/`Wait`.
- `internal/pty/pty.go:124-145` — `Close()` always tries `p.cmd.Process.Kill()` then `p.cmd.Wait()` if `cmd != nil`. **Bug**: on Windows path `cmd` is `nil` (process is created via direct `CreateProcessW`), so `Close` skips killing the child. Need a parallel kill path for `procHandle != 0`. Confirmed by `pty.go:147-180` which handles `Wait` separately for Windows but `Close` doesn't.
- `internal/pty/pty.go:147-180` — `Wait()` correctly forks Unix vs Windows path. Note that `if err != nil { ... }` only sets `exitCode` from `*exec.ExitError`; on success path `exitCode` stays at zero-value (0), which is fine but worth a comment.
- `internal/pty/pty_unix.go:1-58` — clean `creack/pty` wrapper. `waitPlatform()` returns `-1, error` ("not used on Unix") which is correct given `Wait()` branches on `cmd != nil` first.
- `internal/pty/pty_windows.go:1-228` — direct ConPTY via `kernel32.dll`. Uses `unsafe.Pointer` to pack `COORD` into a `uintptr` (`coordToDWORD`, `:34-37`). Endianness assumption (`little-endian x86/ARM`) is documented but should be asserted at compile time or via a build tag — Windows-on-ARM64 is little-endian so it's actually fine, but this code assumes Win32-compatible byte layout.
- `internal/split/`, `internal/tabmgr/` — each ships with `*_test.go` peers (4 test files total). Couldn't audit all 3175 diff lines in this slice; the pty layer is the riskiest section.

## Verdict: `request-changes`

The package boundaries are clean and the cross-platform abstraction is well-shaped. But three concerns:

1. **`Close()` does not terminate the Windows child process** (`pty.go:124-145`). The `if p.cmd != nil` branch is the only place that kills; on Windows `cmd` is always nil because `pty_windows.go` creates the process via `CreateProcessW` directly. Need an `else if p.procHandle != 0 { TerminateProcess(p.procHandle, 1) }` or equivalent. Without this, closing a tab leaks the child shell on Windows.

2. **No goroutine cancellation around `Read`/`Write`.** `pty.go:90-100` exposes `Read(buf)` returning whatever `p.stdout.Read(buf)` returns. After `Close()`, a goroutine blocked in `Read` will only unblock if the underlying handle close races correctly. Should document or add a `context.Context`-aware variant for use from the TUI.

3. **Scope.** This is 3000+ lines of net-new infrastructure with no consumer in the same PR. Recommend splitting: (a) `internal/split/` standalone, (b) `internal/tabmgr/` standalone, (c) `internal/pty/` standalone with a tiny demo command under `cmd/` that exercises it on both platforms in CI. Three smaller PRs are much easier to review and bisect than one 15-file omnibus.

The work is good quality. The Windows-close bug is the only correctness issue I can point at concretely; the rest is process/scope.
