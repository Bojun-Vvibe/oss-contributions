# sst/opencode#24740 — fix(opencode): batch vcs git show calls

- Head SHA: `9d41bf6`
- Author: ualtinok
- Files: 6 / +202 / −6
- Closes #24739

## Summary

Replaces the per-file `git show HEAD:<file>` process fan-out in `/vcs/diff` (which spawned one short-lived `git` per changed file — hundreds for large refactors) with a batched `Git.showMany()` path using a single `git cat-file --batch` process. Adds a debounced VCS refresh on the file-watcher side and a normalized `.git`-metadata path filter so a burst of file events doesn't multiply diff requests. Keeps the per-file `Git.show` path as a correctness fallback when the batch input would be malformed.

## Specific observations

- Right-shape primary fix at `packages/opencode/src/git/index.ts:194-270` — `showMany` runs `git cat-file --batch` with one stdin line per `${ref}:${prefix}${file}` target, parses the binary `<sha> blob <size>\n<content>\n` framing back into a `Map<file, content>`, and falls back to per-file `show` with `concurrency: 8` if any input contains `\n`/`\r` (which would corrupt the line-delimited batch protocol). The fallback-on-malformed-input guard at `:213` is the load-bearing safety net — without it, a filename with embedded newline would wedge the batch parser silently.
- Watcher-side coalescing at `packages/app/src/pages/session.tsx:621` — `watcherRefreshVcs = createDebouncedCallback(refreshVcs, 100)` paired with `onCleanup(watcherRefreshVcs.dispose)` so the debounce timer doesn't leak across route changes. The 100ms window is short enough to feel real-time but long enough to coalesce a typical save-burst.
- New `isGitMetadataPath` helper at `helpers.ts:175-178` normalizes Windows backslashes (`replaceAll("\\", "/")`) and matches `.git`, `*/.git`, `.git/*`, and `*/.git/*`. Negative-case test at `helpers.test.ts:170-173` pins both `src/git/index.ts` (literal `git` in path component) and `src/.github/workflows/test.yml` (the GitHub-Actions config trap that a naive `.git` substring match would catch) — these are the two cells that prevent silent over-filtering of legitimate user file events.
- Debounce semantics test at `helpers.test.ts:122-138` with `vi.useFakeTimers()` advances 99ms → 0 calls, +1ms → 1 call, exact boundary check. Dispose-cancels-pending test at `:140-156` pins the cleanup contract.
- The pre-existing `isGitMetadataPath` matcher uses `.endsWith("/.git")` which doesn't catch `.git` followed by Windows backslash inside a non-normalized path — but the `replaceAll("\\", "/")` upfront normalization handles that. Order-of-operations is correct.

## Verdict

`merge-as-is`

## Rationale

Real performance fix backed by a measurable workload (large-refactor diff = N processes → 1 process). The batch implementation correctly preserves the per-file fallback path so correctness is never sacrificed for speed. Test coverage is thorough across three dimensions (debounce timing, path-normalization positive/negative, batch fallback via existing `git.test.ts`). The `vi`-from-`bun:test` import at `helpers.test.ts:1` is unusual but the rest of the file already uses `bun:test` so it's consistent with the codebase. No nits material enough to block — the batch protocol parser is the only place I'd want to see a fuzz test in a follow-up, but the fall-through-to-per-file safety net makes that low-risk.

