# Review — openai/codex#20892 — feat(tui): add PR summary statusline items

- PR: https://github.com/openai/codex/pull/20892
- Head: `9fcff3ddb171b290615cf48a319f1b01a8d03372`
- Verdict: **merge-after-nits**

## What the change does

Adds a `branch_diff_stats_to_default_branch` helper to
`codex-rs/git-utils/src/info.rs` returning a new `GitBranchDiffStats { additions,
deletions }` struct, plus a private `DefaultBranch` discovery type. The flow:
`get_default_branch` → `git merge-base HEAD <merge_ref>` → `git diff --numstat
<base>..HEAD`, summed across files. Tests in
`codex-rs/core/src/git_info_tests.rs` cover (a) clean branch returns 0/0 and
(b) feature branch with one modified + one new file reports 3 additions / 1
deletion.

## Strengths

- Uses the existing `run_git_command_with_timeout` so the new call inherits
  the timeout / cancellation behavior — no risk of hanging the TUI on a
  pathological repo.
- Explicit early returns on every git failure path
  (`merge_base.status.success()`, empty `merge_base`, parse failure) → the
  helper returns `None` rather than panicking, which is correct for a
  status-line widget.
- Tests rename the default branch to `main` before asserting, which makes the
  test independent of git's `init.defaultBranch` config on the CI host.

## Nits / asks

1. `additions += columns.next().and_then(...).unwrap_or(0)` silently treats
   `-` (binary file marker in numstat) as 0. That's defensible but worth a
   comment so a future reader doesn't think it's a parse bug.
2. `branch_diff_stats_to_default_branch` does **three** git invocations
   (`rev-parse` via `get_git_repo_root`, `merge-base`, `diff --numstat`)
   per status-line refresh. If the TUI calls this on a tick, consider an
   in-process LRU keyed on `(cwd, HEAD sha, default_branch sha)`.
3. `GitBranchDiffStats` derives `Eq` but stores `u64` — fine; just confirm
   downstream serialization (if any) doesn't lose precision when crossing
   into the TS app-server boundary as `number`.

Approve once the binary-file-marker comment lands.
