# google-gemini/gemini-cli #26330 — fix(cli): ensure branch indicator updates in sub-directories and worktrees

- **Head SHA:** `b36b5ffe72c535993e701dc64e9a28db9bfd014d`
- **Files:** `package-lock.json` (+32 / -3, all `"peer": true` annotations), `packages/cli/src/ui/hooks/useGitBranchName.test.tsx` (+122 / -56), `packages/cli/src/ui/hooks/useGitBranchName.ts` (+35 / -18), `packages/core/src/utils/gitUtils.ts` (+12)
- **Verdict:** `merge-after-nits`

## Rationale

Real bug, focused fix. The branch indicator in the TUI was watching the wrong path (parent repo root rather than the current working directory's git context), so users in subdirectories or in linked worktrees saw a stale branch. The fix moves resolution into `gitUtils.ts` (+12 lines for what looks like a worktree-aware `getBranchName(cwd)`) and updates `useGitBranchName.ts` (+35/-18) to consume it; the test file gains 122 lines of new coverage, replacing 56 lines of previous mocks — a healthy ratio.

The `package-lock.json` churn is non-functional: it only adds `"peer": true` flags to ~30 dep entries (`@bufbuild/protobuf`, `@grpc/grpc-js`, OpenTelemetry packages, `acorn`, `express`, etc.). This is what npm 11 emits when a transitive boundary changes; safe but unrelated to the bug. Consider asking the author to split the lockfile churn into a separate `chore: refresh package-lock` commit so the substantive diff is easier to bisect, but it's not blocking.

Nit: confirm `getBranchName` handles the worktree case by reading `.git` as a file (gitdir pointer) rather than assuming `.git/HEAD`. With that confirmed, this is a clean merge.

