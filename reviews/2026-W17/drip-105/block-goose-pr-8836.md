---
pr: https://github.com/block/goose/pull/8836
sha: bcc29550
diff: +14/-3
state: OPEN
---

## Summary

Drops the `-i` (interactive) flag from the two login-shell PATH-resolution invocations (`goose-mcp/src/subprocess.rs` and `goose/src/agents/platform_extensions/developer/shell.rs`) so the spawned shell can't call `tcsetpgrp()` to claim the controlling-terminal foreground process group from the parent goose process. With `-i`, goose was getting backgrounded mid-startup and then receiving `SIGTTOU` when `rustyline` later tried to enter raw mode, causing the process to suspend. `-l` (login) is preserved and is what actually sources the user's profile-defined `PATH`.

## Specific observations

- `crates/goose-mcp/src/subprocess.rs:60-65` — the change itself is `args(["-l", "-i", "-c", "echo $PATH"])` → `args(["-l", "-c", "echo $PATH"])`. One flag removed. The five-line comment block immediately above explains exactly the mechanism (`tcsetpgrp` → foreground group steal → SIGTTOU on raw-mode entry). The comment is the load-bearing artifact here — without it, a future contributor will absolutely re-add `-i` thinking it's needed for `.zshrc` / `.bashrc` to be sourced. Comment quality is high.
- `crates/goose/src/agents/platform_extensions/developer/shell.rs:171-189` — same removal, same comment, applied to *both* the flatpak and non-flatpak code paths. Critical to do both: a flatpak user would have hit the bug only inside the sandbox if just one path was patched. Symmetric coverage.
- The justification in the comment is technically precise: `-i` triggers shell rc-file sourcing (`.bashrc` for bash, interactive blocks of `.zshrc` for zsh). For PATH purposes, `-l` already sources the *login* files (`.bash_profile`, `.profile`, `.zprofile`, `.zlogin`) where `PATH` exports conventionally live. There is a real population of users who put `export PATH=...` in `.bashrc` or interactive `.zshrc` blocks rather than the login files — those users will see different (smaller) `PATH` results after this change. That's a behavior regression for them. The PR doesn't mention this trade-off; it should, with a recommended workaround ("move PATH exports to `.bash_profile`/`.zprofile` if you relied on `-i` sourcing them").
- The dotfile-sourcing-norms argument cuts both ways: macOS Catalina+ ships zsh as default and the modern convention is `.zprofile` for PATH, but plenty of legacy bash users have `.bashrc`-only PATH exports. A diagnostic improvement worth considering: if `resolve_login_shell_path()` returns a `PATH` shorter than `env::var("PATH")`, log a warning saying "your login shell didn't add to PATH; if you set PATH in `.bashrc`/`.zshrc`, move it to a login file." That converts the silent regression into an observable one.
- No tests added or modified. SIGTTOU / process-group bugs are notoriously hard to integration-test (you'd need a controlling terminal in CI), so this is forgivable, but a unit test that asserts the args list contains `-l` and does *not* contain `-i` would at least guard against accidental regression of this fix:
  ```rust
  // pseudo: collect args before spawn, assert.
  ```
- The two patched files have identical comment blocks (5 lines, same phrasing). If these duplications drift later, the inconsistency itself becomes confusing. Consider extracting `resolve_login_shell_path` into a shared module — the function name is identical in both files and they likely have ~70% overlap. Out of scope for a SIGTTOU fix, but worth a follow-up issue.
- The reproduction signal in the PR description ("goose suspends on startup") is very specific and the root-cause analysis is correct. SIGTTOU on background-process raw-mode-entry is a real phenomenon (it's why `less` and `vim` both go to enormous lengths to manage their own process group); this is the right fix for it.

## Verdict

`merge-after-nits` — correct fix for a real foreground-group-stealing bug, with comments good enough that the next contributor won't undo it. Two follow-ups: (1) call out in PR description / release notes that users who export PATH from `.bashrc` or interactive `.zshrc` blocks will see a different (login-shell-only) PATH after this change, with the recommended remediation; (2) optional: add a one-liner regression test asserting the args list shape.

## What I learned

`-l` and `-i` look interchangeable for "I want my user's environment" but they pull in completely different files (login vs. interactive rc) and `-i` carries a hidden side effect: an interactive shell expects to own the terminal foreground group, and on POSIX it will take it if it can. Spawning an interactive shell as a child of a TUI is a recipe for SIGTTOU/SIGTTIN deadlocks that surface as "my app suspends mysteriously on startup." The fix is almost always "don't pass `-i`."
