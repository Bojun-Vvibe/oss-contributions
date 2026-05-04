# openai/codex #21001 — feat(tui): route /diff through workspace commands

- **Head SHA:** `df1aaba90a8b9ee9cf835d8dcf93929b762b72f5`
- **Size:** +323 / -78 across 3 files
- **Verdict:** **merge-after-nits**

## Summary
Migrates the TUI `/diff` slash command from direct local `tokio::process::Command`
git invocations to the new `WorkspaceCommandExecutor` abstraction introduced by the
stacked dependency #20892. This unblocks remote app-server sessions where the active
workspace is not on the same host as the TUI.

## Strengths
- `slash_dispatch.rs` (around line 325) cleanly captures the runner clone and the
  fallback `cwd` (uses `current_cwd` if set, else `config.cwd`) before the spawn,
  which keeps the future `'static` and avoids accidental capture of `&self`.
- `get_git_diff.rs` rewritten to take `&dyn WorkspaceCommandExecutor` plus an
  explicit `cwd: &Path` is the right shape — explicit is better than the previous
  implicit-CWD inheritance for a remote-friendly model.
- Adds `DIFF_COMMAND_TIMEOUT = 30s` constant — sane default; matches the 30s that
  existing workspace commands tend to use.

## Nits
- The error path when `runner` is `None` returns the literal string
  `"Failed to compute diff: workspace command runner unavailable"` to the user.
  This is technically a programmer error (every `ChatWidget` instantiated through
  the stacked branch should have a runner). Worth either upgrading to a `tracing::error!`
  + a friendlier user message, or asserting at construction time that the runner
  exists.
- `get_git_diff` now returns `Result<_, String>` instead of `io::Result`. Stringly
  typed errors lose structure for callers that may want to distinguish timeout from
  upstream git failure. A small enum (`GitDiffError::{Timeout, Runner(String), GitExit(i32)}`)
  would scale better, but acceptable for now since the only consumer is the TUI.
- Stacked on #20892 — verify the merge order or rebase before landing so reviewers do
  not have to mentally diff against an unmerged base.

## Recommendation
Land after a `tracing::error!` for the missing-runner branch and confirmation the
stacked base merged cleanly. Error-typing cleanup can be a follow-up.
