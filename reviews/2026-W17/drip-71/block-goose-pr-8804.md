# block/goose PR #8804 — prevent login-shell PATH probe from suspending goose on startup

- **PR**: https://github.com/block/goose/pull/8804
- **Author**: @maxamillion (Adam Miller)
- **Head SHA**: `c59fcb8f49d11ae6c7994c36aa35dfb9363c2047`

## Summary

`crates/goose/src/agents/platform_extensions/developer/shell.rs` —
the `resolve_login_shell_path()` function spawns a `bash -l -i -c
'echo $PATH'` subprocess to read the user's full login PATH. When
goose is launched from a TTY, the interactive bash subprocess called
`tcsetpgrp(TIOCSPGRP)` to claim the foreground process group, which
then sent `SIGTTIN`/`SIGTTOU` to goose itself — suspending the parent.

Fix: spawn the subprocess in a new session via the `process-wrap`
crate's `ProcessSession` wrapper, which calls `setsid()` so the child
cannot grab the controlling terminal of the parent.

```rust
use process_wrap::std::{CommandWrap, ProcessSession};
...
cmd.wrap(ProcessSession);
let mut child = cmd.spawn().ok()?;
```

Adds the dependency `process-wrap = { version = "9.1.0", features = ["std"] }`
in `crates/goose/Cargo.toml:196`.

## Verdict: `merge-as-is`

Correct fix for a textbook job-control bug. The inline comment at the
`cmd.wrap(ProcessSession);` call site even names the actual
mechanism (`TIOCSPGRP` → `SIGTTIN`), which is exactly what a future
maintainer needs.

## Specific references

- `shell.rs:166-189` — restructured to build a `CommandWrap` then
  apply `ProcessSession`. The flatpak vs direct branch is now just
  about the inner `Command`; both arms get the same `wrap()` call.
- `shell.rs:206` — `child.stdout()` (note: method, not field) — the
  `process-wrap` `WrappedChild` exposes stdout via accessor.
- `shell.rs:218-220` — the failure path `let _ = child.kill();`
  drops the `wait()` follow-up. Minor concern below.
- `Cargo.lock:4391` and `Cargo.toml:196` — dependency wired in cleanly.

## Risks

1. **`child.wait()` no longer called on the kill-fallback path.**
   The previous code did `child.kill(); let _ = child.wait();` after
   the timeout. The new code only calls `kill()`. With
   `process-wrap`'s `WrappedChild`, the `Drop` impl probably handles
   reaping, but worth confirming — orphaned zombies on macOS would
   only show up under sustained load. Either restore the `wait()` or
   document that `WrappedChild::Drop` covers it.
2. **`ProcessSession` semantics on macOS vs Linux.** Both should
   call `setsid(2)`, but the `process-wrap` README's compatibility
   matrix should be linked in the PR description so reviewers can
   confirm. Flatpak case (where the inner command is
   `flatpak-spawn`) needs especially careful checking — flatpak may
   or may not honor `setsid` from outside the sandbox.
3. **New transitive deps.** `process-wrap` 9.x pulls in `nix`,
   `tracing`, possibly `winapi` features. Cargo.lock churn is
   contained (one new direct entry visible) but a `cargo tree -d`
   output in the PR would close the supply-chain question.

## Nits

- The `s: std::process::ExitStatus` type annotation on the `is_ok_and`
  closure (line 207) is needed because `WrappedChild::wait` doesn't
  return the bare `ExitStatus`. Worth a comment so it doesn't get
  "cleaned up" later.
