# sst/opencode #25773 — fix(desktop): preserve shell PATH for sidecar

- URL: https://github.com/sst/opencode/pull/25773
- Head SHA: `07fa4132eb9fbd7de738bacb41743dea921aed07`
- Author: Hona
- Size: +10 / -7 across 1 file

## Comments

1. `packages/desktop/src-tauri/src/cli.rs:447` — Switching from a string-based `ends_with("/nu")` check to a typed `is_nushell(&shell)` helper is the right shape, but the helper isn't shown in the diff. Confirm `is_nushell` lives somewhere in the same file/module and is `#[inline]`-able; if it just wraps the same `ends_with` check it's a wash, but if it actually parses the shell name (handling `nu.exe` on Windows, symlinks, etc.) that's a meaningful improvement worth a one-line comment.
2. `packages/desktop/src-tauri/src/cli.rs:449-453` — Nushell branch keeps the previous behavior (spawn the shell with `-l -c "<line>"`). The `-l` (login shell) flag is critical here because that's what causes nushell to source the user's env config and thus inherit their PATH. Good — preserved.
3. `packages/desktop/src-tauri/src/cli.rs:454-459` — Non-nushell branch now spawns the sidecar **directly** with `Command::new(sidecar)` instead of going through the shell. This is the actual fix: `merge_shell_env(load_shell_env(&shell), envs)` already populated `envs` with the user's interactive PATH (from a prior `shell -lc 'env'` round-trip), so the child process gets the right env without needing the shell-as-launcher hop. Cleaner and avoids quoting hazards.
4. `packages/desktop/src-tauri/src/cli.rs:457` — `args.split_whitespace()` is a real correctness regression for any argument containing spaces or quotes. The previous shell-mediated path passed `args` as a single shell line so `\"path with space\"` would survive; the new direct-spawn path will tokenize `--prompt "hello world"` into four args. **This needs to change** — either accept `args: &[String]` from the caller and skip splitting entirely, or use `shlex::split` and propagate the parse error.
5. `packages/desktop/src-tauri/src/cli.rs:447-460` — The two-arm `cmd` initialization with mutable rebinding inside `let mut cmd = if ... { ... } else { ... }` is fine but a `match shell_kind { Nushell => ..., Other => ... }` with a small enum reads cleaner and pairs naturally with `is_nushell`.
6. Missing: a test (or at minimum a manual repro note in the PR body) for the original bug — "launching from Finder vs. from a terminal produces different PATH" is the classic macOS .app bundle issue. A regression test that snapshots the `envs` map after `merge_shell_env` would prevent future shell-handling refactors from re-breaking this.
7. Windows path: `is_nushell` likely returns false on Windows (where the user's shell may be `pwsh.exe` or `cmd.exe`); the direct-spawn branch will then run the sidecar without any shell environment expansion. Confirm the desktop app already handles Windows PATH inheritance through Tauri's own env machinery, or this fix only helps macOS/Linux.

## Verdict

`request-changes`

## Reasoning

The architectural change (spawn sidecar directly when not nushell) is correct and the right fix for the macOS PATH-from-Finder problem. However, `args.split_whitespace()` on line 457 is a real regression for any argument containing spaces — that needs to be replaced with proper shell tokenization (or the caller signature changed to pass a pre-split `Vec<String>`). Once that's fixed and there's a smoke test for a spaced argument, this is mergeable.
