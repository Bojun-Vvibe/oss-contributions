# sst/opencode#25211 — fix(tui): scope Zed editor context to containing workspaces

- Link: https://github.com/sst/opencode/pull/25211
- Head SHA: `73bbf2a2a47b93b96dce1a393d8838ec1155494d`
- Files: `cli/cmd/tui/context/editor-zed.ts` (+10/−3), `test/cli/tui/editor-context-zed.test.ts` (+86/−2)
- Fixes: #25164

## Notes
- Two-pronged fix at `editor-zed.ts`: (a) wraps `Filesystem.stat(item)?.isFile()` in a `try/catch isFile` helper at `:194-201` so an `ELOOP`/`EACCES` on a single candidate (symlink loop, sandboxed FS without the user `~/.local/share/zed` path) returns `false` instead of throwing and aborting the entire TUI bootstrap, (b) replaces the symmetric `pathContains(a,b) || pathContains(b,a)` workspace-scoring at `:204-205` with a one-direction `pathContains(item, cwd) ? path.resolve(item).length : 0` ranking that *only* matches when the Zed workspace contains the session cwd, and uses path-length as the tiebreaker so the most-specific ancestor wins (`/repo/packages` ranks above `/repo` for cwd `/repo/packages/app`). Eliminates the bug where a Zed workspace nested *inside* the session dir (e.g. an effect-lab subdir the user opened in Zed once) hijacks the editor context for an unrelated parent-dir session.
- The `path.resolve(item).length` ranking at `:206` is the right tiebreaker: post-`resolve` it's normalized to absolute form so two equivalent paths get the same score, and longer = more specific in the containing-ancestor sense. Score `0` (no containing match) correctly suppresses the prior fallback that returned a nested-workspace result.
- Three new test arms pin the regression contract: `resolveZedSelection matches a Zed workspace that contains the session directory` at `:273-292` (positive case for `/tmp/packages/app` cwd against `/tmp` workspace), `prefers the most specific containing Zed workspace` at `:295-326` (two-row fixture: `/tmp` workspace ranks 4 chars, `/tmp/packages` ranks 13 chars → child wins), and `ignores a Zed workspace nested inside the session directory` at `:329-336` (the load-bearing anti-behavior arm: workspace `/tmp/effect-lab` against cwd `/tmp` returns `{type: "empty"}` rather than the nested workspace's selection — pre-fix this returned the nested selection).
- The `resolveZedDbPath skips candidates that cannot be stated` test at `:71-86` uses `await symlink(loop, loop)` to create a self-referential symlink and asserts `resolveZedDbPath()` returns `undefined` rather than throwing, which is the right end-to-end test of the new `try/catch` arm.

## Nits
- `path.resolve(item).length` for tiebreaker is correct but length-as-specificity is fragile against symlinks pointing to deeper paths or `..`-laden inputs that resolve longer than they look — `path.resolve(item).split(path.sep).length` (segment count) would be more robust against pathological inputs and equally cheap.
- The `isFile` helper at `:194-201` swallows *any* exception silently — a permanently-unreadable Zed DB on a misconfigured host would become invisible. A one-line `Log.create("editor-zed").debug({err}, "skipping zed db candidate")` inside the catch would aid diagnosis without changing behavior.
- The two `path.resolve(item)` calls inside `scoreZedWorkspace`'s `.reduce` happen for every workspace path on every call — caching `pathContains`'s already-resolved form (or hoisting the `resolve` outside the reduce) would avoid the O(N²)-ish redundant resolves on a user with many Zed workspaces.
- No test covers the `Math.max(score, ...)` aggregation path when one workspace contains cwd at score 13 and another contains it at score 27 — the existing `prefers the most specific` test covers the two-workspace case but not the three-workspace tie-break-then-replace path.

**Verdict: merge-after-nits**
