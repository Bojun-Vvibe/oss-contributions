---
pr: 2652
repo: charmbracelet/crush
sha: 51423563a79cc81e99afa4e6561f7892c9132163
verdict: merge-as-is
date: 2026-04-27
---

# charmbracelet/crush #2652 — fix(grep): stop regex fallback after cancellation

- **Head SHA**: `51423563a79cc81e99afa4e6561f7892c9132163`
- **Size**: small, single-file change in `internal/agent/tools/grep.go`

## Summary
Threads `context.Context` through the regex-fallback grep path so a user-cancelled grep doesn't continue burning CPU walking the file tree after `ripgrep` failed. Adds eight `if err := ctx.Err(); err != nil { return … }` checks at every meaningful loop boundary in `searchFiles`, `searchFilesWithRegex`, `fileContainsPattern`, and within the `filepath.Walk` callback.

## Specific findings
- `grep.go:175-177` — `searchFiles` now checks ctx before calling `searchWithRipgrep`. Cheap guard; fine.
- `grep.go:181-183` — re-checks ctx *between* the ripgrep failure and the regex fallback. This is the actual fix: previously, ctx-cancellation while ripgrep was running would surface as a ripgrep error, and the code would then try the regex fallback regardless. Now the cancellation propagates correctly.
- `grep.go:184` — `searchFilesWithRegex` signature changed to take `ctx context.Context` as first param. Consistent with idiomatic Go.
- `grep.go:272-275` — fallback re-checks ctx at entry. Defensive but cheap.
- `grep.go:301-303` — inside the `filepath.Walk` callback, returns `ctx.Err()` if cancelled. `filepath.Walk` correctly propagates this as the walk's terminal error. Note: the existing pattern at `grep.go:298` already does `if err != nil { return nil // Skip errors }` — the new ctx check at `:301-303` is *after* that, so the order is right (ctx-cancellation propagates, file-stat errors get skipped).
- `grep.go:328-330` — re-checks ctx inside the `fileContainsPattern` error branch. Subtle but right: a `Open()`-returns-EACCES error would normally be silently skipped via `return nil`, but if ctx is also cancelled the cancellation wins.
- `grep.go:355-357` — final ctx check after the whole walk completes. Belt-and-suspenders; fine.
- `grep.go:362-365` — `fileContainsPattern` signature changed to take `ctx`. Entry-point check before the `isTextFile` test (which probably doesn't allocate but has a few syscalls).
- The pattern is consistently `if err := ctx.Err(); err != nil { return …, err }` rather than naked `select { case <-ctx.Done() … }`. Cheaper for a tight loop and idiomatic for tools that can't block on a channel because they're already in a callback. Good.
- No test added but the cancellation behavior is hard to unit-test deterministically; the change is mechanical and audit-by-eye is reasonable.

## Risk
Very low. Pure additive cancellation propagation; no behavior change in the success path.

## Verdict
**merge-as-is** — textbook ctx-propagation fix, mechanical, consistent style, every meaningful boundary covered.
