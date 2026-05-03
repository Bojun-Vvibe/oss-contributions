# Review: google-gemini/gemini-cli PR #26330

- **Title:** fix(cli): ensure branch indicator updates in sub-directories and worktrees
- **Author:** Adib234
- **Head SHA:** `b36b5ffe72c535993e701dc64e9a28db9bfd014d`
- **Verdict:** request-changes

## Summary

Fixes #19271: the footer branch indicator does not update when the
user runs `!git checkout <branch>` from a CLI sub-directory or inside
a git worktree. Root cause: `useGitBranchName` watched
`./.git/logs/HEAD` relative to CWD, which doesn't exist outside the
repo root or for worktrees. PR resolves the absolute git dir via
`git rev-parse --absolute-git-dir` and watches `<gitDir>/HEAD`
instead. New helper `getAbsoluteGitDir` lives in
`packages/core/src/utils/gitUtils.ts`. Tests rewritten with fake
timers and `getAbsoluteGitDir` mocked.

## Specific-line comments

- `packages/core/src/utils/gitUtils.ts:+12` — small new helper. Diff
  excerpt is cut off; it should run `git rev-parse --absolute-git-dir`
  via `spawnAsync` and return stdout trimmed, returning `undefined`
  on non-zero exit. Verify error handling is silent (no spam in
  non-git directories).
- `packages/cli/src/ui/hooks/useGitBranchName.ts:+35/-18` — the watch
  target switch from `.git/logs/HEAD` to `<absoluteGitDir>/HEAD` is
  the right fix. `HEAD` updates atomically on `git checkout` (write +
  rename), so `fs.watch` will fire reliably. `logs/HEAD` only updates
  on commits/checkouts that produce reflog entries — actually a
  superset of what's needed, but the file may not exist in shallow
  clones or with `core.logAllRefUpdates=false`.
- `packages/cli/src/ui/hooks/useGitBranchName.test.tsx:46-72` — test
  rewrite uses `vi.useFakeTimers()` and a deferred-spawn pattern.
  Reasonable, but the deferred shape (`code: number` added to the
  resolved value) suggests the production helper now consumes
  `code` — make sure that propagates back through `spawnAsync`'s
  return type without a parallel type drift.
- **`package-lock.json` is in the diff with 32 added / 3 deleted, all
  of which are `"peer": true` annotations on first-party packages.**
  That is not from this PR's source change — it's drift from a
  different `npm install` flow (likely npm 11+ recomputing peer
  resolutions). It will conflict with concurrent PRs and is unrelated
  to the branch-indicator fix. **This must be reverted.**

## Risks / nits

- No worktree-specific test. The PR claims worktree validation in the
  Pre-Merge Checklist, but there's no automated test that mocks
  `git rev-parse` returning an out-of-tree path (e.g.
  `/repo/.git/worktrees/feature-x`) and asserts the watcher attaches
  there. The whole point of the fix is worktrees — please add one.
- `git rev-parse --absolute-git-dir` is a child process. Spawning it
  on every render of the hook is wasteful; verify it's gated by
  `useEffect` with stable deps (the diff excerpt doesn't show this).
- `fs.watch` on macOS has known issues with atomic rename; depending
  on how the watcher is set up, `git checkout` (which uses
  write+rename) may not fire on macOS. The previous logs/HEAD-append
  approach side-stepped this. Worth a manual test on macOS before
  merging.

## Verdict justification

The substantive fix is correct and the right shape. But the
unrelated `package-lock.json` peer-true churn is a real concern: it
will conflict with every concurrent PR and is not justified by this
change. **request-changes** to revert the lockfile noise and add a
worktree-specific test; the `useGitBranchName` change itself can land
as-is once those are addressed.
