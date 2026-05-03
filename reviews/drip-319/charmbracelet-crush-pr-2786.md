# Review: charmbracelet/crush #2786 — fix: account for section overhead in sidebar height allocation

- PR: https://github.com/charmbracelet/crush/pull/2786
- Head SHA: `0e1e099e179cd7d4111d72f376ff62f7f49a0ee7`
- Author: mkaaad
- Files touched: `internal/ui/model/sidebar.go` (+5 / -1)

## Verdict: `merge-after-nits`

## Rationale

The fix changes `sidebar.go:169` from a magic `- 6` to a named `sectionOverhead = 11` constant with a justifying comment ("4 section titles, 4 blank lines from `\n\n`, 3 separators = 11") and wraps the result in `max(0, ...)` to prevent underflow on tiny terminals. Both the bookkeeping and the guard are clear improvements over the original. The accounting is plausible from the comment alone but I can't verify it without seeing how `getDynamicHeightLimits` partitions sections — if the section *count* is dynamic (e.g. files section hidden when `len(sessionFiles)==0`), then `11` is an upper bound that will under-allocate when fewer sections render. Looking at the surrounding code: `filesCount := 0` and the loop right after suggest the files section can be empty, so when files=0 the actual overhead is closer to 8 (3 titles, 3 blanks, 2 separators), making the allocation needlessly stingy in that case. Nits: (1) consider deriving overhead from the actual rendered section count rather than hard-coding 11; (2) the `max(0, ...)` requires Go 1.21+ — confirm the project's `go.mod` minimum supports it (built-in `max` was added in 1.21). The change is strictly better than the magic number that came before, so merge after either documenting the Go version requirement or making the overhead dynamic.