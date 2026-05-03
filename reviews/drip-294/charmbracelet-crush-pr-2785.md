# charmbracelet/crush PR #2785 — fix(tools/view): limit view size checks to returned content

- **Repo:** charmbracelet/crush
- **PR:** #2785
- **Head SHA:** `fa1acff88d05871ee16240322f5d818acf08c0ef`
- **Author:** taoeffect
- **Title:** fix: limit view size checks to returned content
- **Diff size:** +208 / -15 across 3 files
- **Drip:** drip-294

## Files changed

- `internal/agent/tools/view.go` (+35/-10) — removes the up-front `fileInfo.Size() > MaxViewSize` reject for non-skill text files (was at the old line ~165) and instead enforces the limit on the *returned section* inside `readTextFile`. New `contentTooLargeError` typed error carries `Size`/`Max`. The `MaxViewSize` check survives for image files only (line 184-188). `readTextFile` gains a `maxContentSize int` parameter; depth-counting in the scanner loop projects byte size as it appends each line and returns the typed error if the projection exceeds the cap.
- `internal/agent/tools/view.md` (+3/-3) — docstring updated: removes BMP/SVG from supported images (interesting side change, see below) and rewords "Max file size: 200KB" to "Max returned file content section: 200KB after offset/limit are applied".
- `internal/agent/tools/view_test.go` (+170/-2) — updates existing call sites to pass the new `maxContentSize` arg, plus a new `TestViewToolAllowsSmallSectionsOfLargeFiles` that builds a file >MaxViewSize, requests a 1-line slice in the middle, and confirms it returns successfully.

## Specific observations

- Core idea is right and overdue: the previous behavior refused to let an agent read line 5 of a 1MB log file, which is a bad UX. Gating on returned content size, not file size, is the correct invariant.
- `view.go:296-308` — the per-line `projectedSize` computation includes `+1` for line separator when `len(lines) > 0`. That correctly accounts for the join newline. Good.
- `view.go:303-305` — when projection exceeds the cap, returns a typed `contentTooLargeError{Size: projectedSize, Max: maxContentSize}` and bails. The caller at `view.go:201-206` does `errors.As(err, &tooLarge)` and emits a `NewTextErrorResponse`. Clean error-typing pattern, idiomatic Go.
- `view.go:200` — `maxContentSize := MaxViewSize; if isSkillFile { maxContentSize = 0 }`. The "0 means unlimited" sentinel is fine but undocumented in the new function signature. Add a one-line `// maxContentSize <= 0 disables the limit` doc comment on `readTextFile`.
- `view.go:285` — `lines := make([]string, 0, min(limit, DefaultReadLimit))` is a smart capacity cap (was `make([]string, 0, limit)` before, which would over-allocate when callers pass huge `limit` values).
- `view.md:1-7` — docs drop BMP and SVG from the supported-images list. Looking at the actual `getImageMimeType` function isn't in this diff, so I can't verify that BMP/SVG were ever truly supported. If they were, this is a silent capability removal masquerading as a docstring fix and should be a separate PR. If they were never supported (the doc was wrong), call that out in the commit message.
- `view_test.go:91-127` — `TestViewToolAllowsSmallSectionsOfLargeFiles` is exactly the right regression test: builds a file with a >MaxViewSize first line, then asks for `Offset: 1, Limit: 1` (the second line, which is small) and asserts success. Pins the new behavior precisely.
- Missing test: a case where `Offset: 0, Limit: 1` on a file with a single >MaxViewSize line *should* still be rejected (or truncated via `MaxLineLength`). The existing `TestReadTextFileTruncatesLongLines` covers the truncation, but worth one explicit assertion that the new content-size cap fires when the section itself exceeds MaxViewSize.
- The `min(limit, DefaultReadLimit)` requires Go 1.21+; presumably already the project's MSRV but worth confirming.

## Verdict: `merge-after-nits`

Correct fix, well-tested. Ask: (1) document the `maxContentSize <= 0` sentinel, (2) clarify whether the BMP/SVG doc removal reflects a real capability change (it shouldn't sneak through here), (3) add a test that the content-size cap fires when a single returned section legitimately exceeds it.
