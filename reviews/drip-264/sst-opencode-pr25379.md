# sst/opencode PR #25379 — feat(worktree): add `.worktreeinclude` support

- **URL**: https://github.com/anomalyco/opencode/pull/25379
- **Head SHA**: `ca0990c7e25def1d6c5948b8c38e036a3bbda892`
- **Files touched**: `packages/opencode/src/worktree/include.ts` (new, ~206 lines), wire-up in worktree creation path

## Summary

Adds opt-in `.worktreeinclude` file at the project root. Each line is an `ignore`-package pattern; matched files (typically gitignored — `.env`, `node_modules/.bin`, build caches) get copied from the source workdir into a freshly-created worktree. Walk uses the existing `FileIgnore.match` to prune build dirs, with a hard-skip on `.git`. Copy honours symlinks and chmod.

## Comments

- `include.ts:42` — `fs.remove(dst, { recursive: true, force: true } as any).pipe(Effect.catch(() => Effect.void))` swallows every error before symlinking. If a stale regular file at `dst` cannot be removed (perm denied), `fs.symlink` will then fail with a less actionable error. Worth narrowing the catch to `ENOENT` only.
- `include.ts:127` — for directories, the matcher only fires when the user pattern includes a trailing slash (`probe = ${rel}/`). The `ignore` package treats `node_modules` and `node_modules/` differently for directory entries; doc this in the file header or normalize patterns so users aren't surprised when `node_modules` (no slash) silently does nothing.
- `include.ts:138` — `FileIgnore.match(rel)` runs on the *relative* path only; if a user pattern is `**/dist`, the recursion will short-circuit at the top-level `dist` before the matcher even sees the deep one. Two-pass (first match, then prune) would be safer than the current interleave.
- `include.ts:165` — reading `.worktreeinclude` with `Effect.catch(() => Effect.succeed(""))` collapses both ENOENT and permission errors into "no patterns". A permission-denied on an existing file should probably warn — silent skip can mask user misconfiguration on CI.
- `COPY_CONCURRENCY = 8` is declared but I don't see it threaded into the `Effect.forEach` in the cut-off snippet — verify the call site actually passes `{ concurrency: COPY_CONCURRENCY }`, otherwise the constant is dead code and copies serialize.
- Naming: `.worktreeinclude` reads close to git's `.gitignore` family but its semantics are inverted (allowlist, not denylist). Consider documenting prominently in the README and a brief example in the file header.

## Verdict

`merge-after-nits` — solid, well-isolated feature with a clean Effect-typed API. The four nits above (symlink-clobber error, directory-pattern semantics, prune ordering, silent read failure) are quality issues, not correctness blockers.
