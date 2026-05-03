# charmbracelet/crush PR #2779 — fix(tools/view): honor limit parameter on files exceeding MaxViewSize

- **Repo:** charmbracelet/crush
- **PR:** #2779
- **Head SHA:** `65b74cd5cdfec06892aa6dde5a44fca2d74fd44a`
- **Author:** pragneshbagary
- **Title:** fix(tools/view): honor limit parameter on files exceeding MaxViewSize
- **Diff size:** +2 / -1 across 1 file
- **Drip:** drip-293

## Files changed

- `internal/agent/tools/view.go` (+2/-1) — at line ~165, changes the size guard from `if !isSkillFile && fileInfo.Size() > MaxViewSize` to `if !isSkillFile && params.Limit <= 0 && fileInfo.Size() > MaxViewSize`. Adds a one-line comment "Only block if no limit is set."

## Specific observations

- `view.go:165-167` — semantically the change is "if the caller bounded the read with `limit`, trust them and let the read proceed even on large files." That's correct and matches the principle that bounded reads can't blow up memory regardless of source file size. Good.
- The guard now relies on downstream code respecting `params.Limit` to not actually read all `fileInfo.Size()` bytes. The diff doesn't show that downstream path — confirm the read implementation uses `io.LimitReader` or seeks/reads only `params.Limit` lines, not "read full file then truncate." If it's the latter, this PR enables OOMs on large files with a small `limit`.
- `params.Limit <= 0` semantics: `Limit == 0` means "default / unset" in this code (treated as no-limit by readers), so blocking on `<= 0` is the right predicate. But verify by reading the type definition / default — if `0` actually means "read 0 lines" somewhere, the guard is too permissive.
- One-line comment ("Only block if no limit is set.") is clear but doesn't capture the *intent* (caller-bounded reads are safe). Worth expanding to one extra line: "When the caller passes an explicit `limit`, the read is bounded regardless of file size, so the MaxViewSize guard becomes unnecessary." That documents the invariant the next maintainer needs.
- No test added. A regression test that views a `MaxViewSize+1` byte file with `params.Limit = 100` and asserts success would pin the new behavior. The codebase has tool tests (e.g. `view_test.go` likely exists) — add one.

## Verdict: `merge-after-nits`

Correct narrowing of an over-eager guard. Two asks before merge: (1) confirm the read path respects `params.Limit` by `io.LimitReader` (not "read all then trim") so large files with small limits don't OOM, (2) add a regression test for `large file + small limit → success`. The fix is right; the safety net needs to be verified.
