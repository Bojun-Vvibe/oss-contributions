# sst/opencode PR #25403 — fix(opencode): prevent /undo from using unsafe root pathspecs

- **PR**: https://github.com/sst/opencode/pull/25403
- **Head SHA**: `af2fbc4f9cdc0b03e403aaba7fab1743ec6ff739`
- **Size**: +49 / −9 across 2 files (`packages/opencode/src/snapshot/index.ts`, `packages/opencode/test/snapshot/snapshot.test.ts`)
- **Verdict**: **merge-after-nits**

## What changed

`snapshot/index.ts` revert pipeline now resolves each candidate file against the worktree root and **rejects** three unsafe shapes before generating `git checkout` pathspecs: paths that resolve outside the worktree (`../`-style), paths that resolve to the worktree root itself, and paths that remain absolute after `path.relative` (Windows drive letters or unrooted resolutions). The dedup key flips from raw `file` to the normalized `rel`, and `git checkout … -- <op.rel>` is now used in both the `single` (lines 408 in the diff context) and batched (line 475) paths instead of `op.file`. A new test `revert ignores worktree root targets` at `snapshot.test.ts:334` exercises the worktree-root case by passing `tmp.path` itself as a file and asserts that the marker file's mtime is unchanged afterward.

## Why this matters

Before this change, a malicious or buggy patch entry containing `state.worktree` itself, an absolute path, or a `../escape/path` would be passed verbatim to `git checkout <hash> -- <path>`. With the worktree as cwd, `git checkout HEAD -- /full/path/to/worktree` is interpreted as a pathspec covering **the entire worktree** — a single bogus entry inside a multi-file revert batch silently widens the blast radius to "revert everything to that snapshot." The new normalize step closes that hole by treating the empty/`.` relative path as a root marker and dropping it, and by rejecting any `rel` that still looks absolute after relativization. Switching the pathspec from `op.file` to `op.rel` is what actually applies the protection — passing absolute paths to `git checkout -- …` is what made root-escape possible in the first place.

## Specific call-outs

- `snapshot/index.ts` line ~382 (`normalize` closure): the three-way check (`outside || root || absoluteRel`) is correct, and skipping unsafe entries with a `log.warn("skipping unsafe revert target", …)` is the right behavior — failing the whole revert would be worse UX. Good defensive coding.
- The Windows drive-letter regex `/^[A-Za-z]:[\\/]/.test(rel)` is necessary because `path.relative` on POSIX won't flag a `C:\…` input as absolute, but pasting Windows paths into a sync'd snapshot is a real concern. Worth keeping.
- Switching `seen` from `Set<string>` of raw `file` to a set of `rel` is a real correctness win: previously `/work/src/a.ts` and `src/a.ts` would dedupe as two entries; now they collapse.
- The new test covers only the worktree-root case. **Nit**: please also add cases for (a) a `../escape` path and (b) a Windows-style `C:\evil\file` value to lock in the other two normalize branches — they're easy to regress on a refactor and the diff already shows you're thinking about all three.
- **Nit**: the `log.warn` call swallows the original raw `file` only — also include `worktree` so postmortem logs explain *why* a path was considered unsafe (otherwise an absolute path inside the worktree looks identical to an escape attempt in the log).

## Verdict rationale

The change is small, well-scoped to a clear vulnerability class, and the test covers the most dangerous case (root expansion). The path-normalization logic is conservative in the right direction — when in doubt, skip. The two nits (extra test cases + richer warn payload) are not blockers but are the obvious follow-ups before this lands. No public API changes, no behavior change for legitimate per-file reverts since rel/file resolve identically for normal entries.
