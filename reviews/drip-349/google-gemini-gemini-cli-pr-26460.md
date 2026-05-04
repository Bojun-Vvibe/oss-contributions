# google-gemini/gemini-cli#26460 — feat(core): allow reusing existing worktrees with --worktree flag

- **URL**: https://github.com/google-gemini/gemini-cli/pull/26460
- **Head SHA**: `7f19202892d9`
- **Diffstat**: +42 / -3
- **Verdict**: `merge-after-nits`

## Summary

Makes `createWorktree(projectRoot, name)` idempotent: if `<projectRoot>/.gemini/worktrees/<name>` already exists and looks like a Gemini-managed worktree, return its path instead of failing on `git worktree add`. Adds path-traversal validation and updates docs to advertise that `gemini --worktree feature-search` resumes an existing worktree.

## Findings

- `packages/core/src/services/worktreeService.ts:120-138` — the path-traversal guard (`relative.startsWith('..') || path.isAbsolute(relative)`) is the right shape but it runs against the path *that `getWorktreePath` already constructed*. If `getWorktreePath` itself sanitizes, this is belt-and-suspenders (fine); if not, also reject `name` containing `/` or `\` early so the error message is intelligible.
- `packages/core/src/services/worktreeService.ts:131-136` — the `fs.promises.stat` + `isGeminiWorktree(worktreePath)` check is the key safety: a same-named directory that *isn't* a Gemini worktree falls through to the create path, where `git worktree add` will fail loudly. Good.
- `packages/core/src/services/worktreeService.ts:135` — `if (err.code !== 'ENOENT') throw err;` — `err` is untyped here; in strict TS this likely needs `(err as NodeJS.ErrnoException).code`. Verify the build passes under `--strict` or guard with a `typeof err === 'object' && err && 'code' in err` check.
- `packages/core/src/services/worktreeService.test.ts:88-110` — new tests cover both branches (does-not-exist → git is called; exists → returns path, git is NOT called). Existing failure test got an explicit `ENOENT` mock so it still hits the create path. Coverage is correct.
- `docs/cli/git-worktrees.md:80-100` — docs now lead with the simple `gemini --worktree feature-search` resume form before the manual `cd` recipe. Reads well.

## Recommendation

Right idempotency model and good tests. Tighten the error typing and confirm path validation is layered correctly, then merge.
