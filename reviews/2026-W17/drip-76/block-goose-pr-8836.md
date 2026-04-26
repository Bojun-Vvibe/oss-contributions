---
pr: 8836
repo: block/goose
sha: bcc2955009e09f5fa05c2228e386b347d08c2c60
verdict: merge-as-is
date: 2026-04-26
---

# block/goose #8836 — fix: drop -i flag from login shell PATH resolution to prevent SIGTTOU

- **URL**: https://github.com/block/goose/pull/8836
- **Author**: michaelneale (Michael Neale)
- **Head SHA**: bcc2955009e09f5fa05c2228e386b347d08c2c60
- **Size**: +14/-3 across 2 files (`crates/goose/src/agents/platform_extensions/developer/shell.rs` + `crates/goose-mcp/src/subprocess.rs`)

## Scope

Removes the `-i` (interactive) flag from the `<shell> -l -i -c "echo $PATH"` invocation in both `resolve_login_shell_path()` call sites:

1. `crates/goose/src/agents/platform_extensions/developer/shell.rs:174` (flatpak path) and `:187` (normal path).
2. `crates/goose-mcp/src/subprocess.rs:60`.

Both sites gain a multi-line comment explaining why interactive mode must not be used.

## Specific findings

- **Root cause analysis is precise and correct.** Interactive shells call `tcsetpgrp()` on the controlling TTY to take ownership of the foreground process group. When goose's child shell does this, the parent (goose CLI) is moved to the background. The next `tcsetattr()` call from rustyline (entering raw mode for the input prompt) triggers `SIGTTOU` because the kernel disallows background-process-group writes to the terminal — and goose suspends with the canonical `suspended (tty output)` shell message. Classic process-group ownership bug; the `-i` removal is the right fix.

- **`-l` alone is sufficient for PATH resolution.** Login shells source `~/.zprofile` / `~/.profile` / `~/.bash_profile` which is where Homebrew and most PATH manipulation lives. The `-i` flag is for sourcing `~/.zshrc` / `~/.bashrc` (interactive-only init), but those rarely contain PATH manipulation that isn't already redundant with the login files. Worth noting the edge case: a user whose `.zshrc` does PATH manipulation but whose `.zprofile` does not (e.g. nvm in `.zshrc` only — common with the official nvm install script) will see a slightly different PATH. The PR's comment doesn't mention this, but the alternative (keep `-i` and live with SIGTTOU) is clearly worse.

- **The comment placement is right** — directly above each `Command::new(&shell).args([...])` chain, with the explanation of *why* the flag is omitted. Future maintainers looking at the args list will see the rationale before they "helpfully" add `-i` back. Both call sites get the comment (slightly different wording in `subprocess.rs:56-60` vs `shell.rs:171-176`); preferable to consolidate to one canonical wording but minor.

- **Both code paths in `shell.rs` (flatpak + normal) are fixed in the same commit.** This matters: a fix that only addresses one path leaves the other broken, and the bug surfaces randomly depending on packaging. The flatpak path uses `flatpak_spawn_process()` which spawns outside the sandbox via `flatpak-spawn --host`, but the host shell still has TTY ownership semantics, so the bug applied there too.

- **`stdin(Stdio::null())` is preserved** at `shell.rs:189` and `subprocess.rs:62`. This is the secondary defense: even if `-i` somehow returned, `stdin: null` should prevent the shell from entering interactive mode for prompt-display purposes. But zsh in particular checks `-i` first and ignores stdin's TTY-ness, which is why the explicit flag removal is the actual fix.

- **`stderr(Stdio::null())` is also preserved** — sensible since this is a probe call where stderr noise would just confuse the operator. Stdout is piped (correct, since that's where `echo $PATH` writes).

- **No regression test added.** Hard to write one — testing process-group ownership in CI requires a real TTY and is platform-specific (Linux + macOS). Manual repro steps are in the PR body and they're correct: build → fresh terminal → `./target/release/goose` → observe banner-then-suspend without the fix, banner-then-prompt with it. Acceptable to land without an automated test.

- **#8631 reference** — the PR cites the regression-introducing commit explicitly. This is the right hygiene: every "drop a flag" fix should name the commit that added the flag so reviewers can read the original justification and confirm the current change doesn't regress whatever motivated the original add. From the PR description, #8631 was "warm shell PATH lookup during session init" — the warm-up itself is good (avoids per-tool-call PATH probes), the `-i` was a cargo-culted addition.

## Risk

Negligible. The worst-case regression is a user whose `.zshrc`-only PATH manipulation no longer applies — but that user can fix it by moving the line to `.zprofile` (the conventional place anyway). Compared to the pre-fix behavior of "CLI suspends on launch", this is an unambiguous improvement. The comment-block additions also make this fix self-documenting against future regressions.

## Verdict

**merge-as-is** — surgically correct fix, both call sites updated in the same commit, comment block prevents the bug from being reintroduced, manual repro steps in the PR body. No tests possible without TTY harness, which is a reasonable trade-off here.

## What I learned

`tcsetpgrp` ownership transfer is the kind of bug that's invisible at the PR-review level when the offending flag is added — `-i` looks like a perfectly innocuous shell argument, and reviewers won't catch the SIGTTOU implication unless they happen to remember it. The defensive lesson is: any `Command::new(shell).args([..., "-i", ...])` in a CLI process that also reads from a TTY needs an inline comment explaining *why* it's safe (or, almost always, getting `-i` removed). Adding this as a clippy lint or a project-level grep would prevent the next regression. The "warm shell PATH lookup" pattern itself is fine — `<shell> -l -c 'echo $PATH'` is the right invocation for it.
