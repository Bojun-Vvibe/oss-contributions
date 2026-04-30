# sst/opencode #25020 — fix(project): stop calling git for linked worktree discovery during startup

- **URL:** https://github.com/sst/opencode/pull/25020
- **Head SHA:** `f4d3934f112b61dbaf4d92564848d2b154e17586`
- **Files:** `packages/opencode/src/project/project.ts`, `packages/opencode/test/project/project.test.ts` (+111/-4)
- **Verdict:** `merge-after-nits`

## What changed

`Project.layer` previously fanned out to `git rev-parse --git-common-dir` and `--show-toplevel` on every startup, even for the common "ordinary repo" case. This refactor:

1. Adds `readLinkedWorktree(dotgit, sandbox)` at `project.ts:175-196` that parses the `gitdir: <admin>` line out of a `.git` file, reads `commondir` + `gitdir` from the admin directory in parallel (`concurrency: 2`), validates the round-trip identity (`linkedGitFile === dotgit`), and returns `{common, worktree, sandbox}` — purely filesystem reads, no subprocess.
2. Adds a `directRepo = dotgitInfo?.type === "Directory"` short-circuit at `:230-261` for ordinary repos: skip `--git-common-dir`/`--show-toplevel` entirely, only run `git rev-list --max-parents=0 HEAD` *iff* the cached `opencode` ID file is missing.
3. Falls through to `readLinkedWorktree` at `:263-302` for the linked-worktree case, again avoiding the topology subprocesses.
4. Falls through to the original `git rev-parse` path only for bare repos / non-standard layouts.
5. Test infrastructure: `mockGitFailure` now accepts `string | string[]` (`project.test.ts:33-44`) and a new `worktree resolves from gitdir metadata without git topology subprocesses` case (`:204-221`) asserts the path resolves correctly even when git refuses `--git-common-dir`, `--show-toplevel`, and `core.bare`.

## Why it's right

- Real perf win on cold-start for the common case (ordinary repo): two subprocess fork+exec calls eliminated unconditionally, plus the `rev-list` call is now skipped when the project ID is already cached.
- The linked-worktree resolution logic correctly mirrors what `git` itself does internally: read `gitdir` pointer → admin dir → read `commondir` + `gitdir` round-trip → validate identity. The `pathSvc.normalize` round-trip check at `:191` is the right defense against symlink/relative-path drift.
- The new test deliberately fails three different git invocations (`--git-common-dir`, `--show-toplevel`, `core.bare`) at once, which locks the no-subprocess contract — if anyone reintroduces a git call on this code path, this test fires.

## Nits

- **Stale `opencode` cache file when worktree commondir moves.** The file write at `:251` does `fs.writeFileString(pathSvc.join(dotgit, "opencode"), id)` for the directRepo case but writes to `linkedWorktree.common` for the linked-worktree case (`:282`). If a user later detaches/reattaches a worktree to a different repo, the `dotgit/opencode` cache could disagree with the `common/opencode` cache. Worth a one-line comment naming the precedence (cache lookup at `:209` reads the `dotgit` path first, so the directRepo cache wins on the same `.git`).
- **Concurrency narrowing.** `Effect.all([...], { concurrency: 2 })` at `:181` is correct but the comment should name *why* — these two reads are independent and on the same admin dir, so kicking them off together saves one filesystem stat-round on cold-page-cache. A future reviewer reading this won't know it's a perf hint vs an ordering-doesn't-matter hint.
- **`stat` on the dotgit path is still a subprocess on some platforms.** `fs.stat` here goes through the `AppFileSystem` Effect adapter — confirm it's a direct `fs.statSync`/`fsPromises.stat` and not shelling out to `stat(1)` or you've replaced two git calls with three syscalls plus a fork. The test fails-git-fast which doesn't catch a hidden subprocess in `fs.stat`.
- **Removed blank line at `:18-20`** is gratuitous churn (right above `const log = Log.create(...)`). Worth restoring to keep the diff focused on the meaningful change.

## Risk

Low-medium. The behavior change is "skip git topology calls when filesystem can answer the same question deterministically", and the linked-worktree branch is exercised by a new test that *also* asserts no git calls happen. The bare-repo / submodule-superproject paths are untouched and still fall through to the old `git rev-parse` code, so the regression surface is bounded to "directRepo-shaped layouts that I missed". Nice ladder of fallbacks.
