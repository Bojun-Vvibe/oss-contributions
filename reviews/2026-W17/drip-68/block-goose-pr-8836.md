# block/goose #8836 — fix: drop -i flag from login shell PATH resolution to prevent SIGTTOU

- **Author:** michaelneale
- **Head SHA:** `bcc2955009e09f5fa05c2228e386b347d08c2c60`
- **Base:** main
- **Size:** +14 / -3 (2 files)
- **URL:** https://github.com/block/goose/pull/8836

## Summary

Regression-fix for #8631 (which warmed the user's shell PATH at session
init by spawning `zsh -l -i -c "echo $PATH"`). The `-i` (interactive)
flag causes zsh to invoke `tcsetpgrp()` and claim the controlling
terminal's foreground process group, demoting the parent goose process
to background. When rustyline subsequently calls `tcsetattr` to enter
raw mode, the now-background goose receives `SIGTTOU` and the kernel
suspends the CLI with `"suspended (tty output)"`. Fix drops `-i` from
both call sites; `-l` alone is sufficient to source
`.zprofile`/`.bash_profile` where PATH is set.

## Specific findings

- `crates/goose-mcp/src/subprocess.rs:59-65` — the call site shifts
  from `.args(["-l", "-i", "-c", "echo $PATH"])` to
  `.args(["-l", "-c", "echo $PATH"])` and gains a 5-line `// NOTE:`
  comment explaining the `tcsetpgrp` mechanism. The comment correctly
  identifies the kernel-level cause (`SIGTTOU` to background pgroup
  attempting `tcsetattr`) and not just the surface symptom.
- `crates/goose/src/agents/platform_extensions/developer/shell.rs:171-177`
  — same comment block prepended above the flatpak branch. The
  comment specifically calls out rustyline as the consumer that triggers
  the failure mode, which is the right reference for the next
  maintainer who wonders why the flag is gone.
- `shell.rs:178-181` — flatpak branch:
  `.args([&shell, "-l", "-i", "-c", "echo $PATH"])` →
  `.args([&shell, "-l", "-c", "echo $PATH"])`. Argv shape preserved
  (still goes through `flatpak-spawn`), only the `-i` flag dropped.
- `shell.rs:185-189` — non-flatpak branch: matching change.
- **Sourcing semantics check (this is the load-bearing claim):** the
  PR claims `-l` alone is sufficient to source profile files where
  `PATH` is set. For zsh: `-l` sources `~/.zshenv` (always),
  `/etc/zprofile` and `~/.zprofile`, then runs the command, then
  `~/.zlogout`. For bash: `-l` sources `/etc/profile`, then the first
  of `~/.bash_profile`/`~/.bash_login`/`~/.profile`. **For most users
  who set PATH in `~/.zshrc` (interactive-shell rc, not profile), the
  PATH won't be picked up by `-l` alone.** The PR description says "PATH
  is configured" in `.zprofile`/`.bash_profile` — true for users who
  follow the documented convention, but a meaningful portion of macOS
  users put PATH exports in `~/.zshrc`. Worth verifying empirically on
  a Mac that ships with the default `~/.zshrc` (per Homebrew and pyenv
  install instructions, which write `eval "$(/opt/homebrew/bin/brew
  shellenv)"` to `~/.zprofile` — so post-Homebrew-install users *are*
  fine, but pre-existing PATH-in-zshrc users may regress).
- `subprocess.rs:60-64` and `shell.rs:171-176` — the comments are
  almost-identical 4–5-line blocks. Minor: factoring this rationale
  into a `///` doc comment on a shared `LOGIN_SHELL_ARGS` constant in
  one place would prevent future drift, but two adjacent identical
  comments in two crates is acceptable for a 14-line fix.
- **No tests added.** The original bug is hard to unit-test (it
  requires a real controlling terminal to reproduce SIGTTOU), but a
  comment in either modified file pointing at issue #8631 and the
  symptom string `"suspended (tty output)"` would help the next
  reader.

## Verdict

`merge-after-nits`

## Reasoning

Diagnosis is correct, fix is the minimum viable patch, and the comment
captures the right kernel-level reasoning so the flag isn't
re-added by a well-meaning future contributor. The reproduction steps
in the PR body are concrete and runnable.

The one nit worth raising before merge: validate that `-l` alone
picks up PATH on the supported macOS shell-config conventions. For
users who only export PATH from `~/.zshrc` (no `.zprofile`), this fix
trades a "suspended" failure for a "missing PATH entries" failure that
will be harder to diagnose because it's silent. A belt-and-suspenders
option is to fall back to reading `getenv("PATH")` if the spawned
shell returns an empty or default-only path — but that's a follow-up,
not a blocker for the SIGTTOU fix which is itself correct.
