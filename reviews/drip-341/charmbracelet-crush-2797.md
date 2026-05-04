# charmbracelet/crush #2797 — fix(ui): style lockup

- **Head SHA:** `cb6eae7e702a31e11990c198c56b7f21d1ae1cbb`
- **Size:** +38 / -4 across 2 files
- **Verdict:** **merge-after-nits**

## Summary
Restores rendering performance regressed by the prior theme-prep overhaul (commit
`755f6fae...`). Memoizes the markdown and quiet-markdown renderers per-width with a
mutex-guarded cache, and exposes `InvalidateMarkdownRendererCache()` to be called
when the active styles change. The motivation: the previous code rebuilt the
glamour `TermRenderer` for every token chunk, which trashed the per-item render
cache and forced a full re-render on each chunk arrival.

## Strengths
- Lock-then-check-then-build pattern in `MarkdownRenderer` and `QuietMarkdownRenderer`
  (around line 25 and 50 of `internal/ui/common/markdown.go`) is correct under the
  single mutex; no double-build race.
- Explicit `InvalidateMarkdownRendererCache()` keyed off `applyTheme` (called in
  `internal/ui/model/ui.go`) is the right wiring — width-keyed cache is invariant
  under theme changes only, and dropping the whole map is cheap.
- Diff is small and surgical for a perf fix, which is what the situation calls for.

## Nits
- A single `sync.Mutex` serializes both renderer fetches across all widths.
  In practice the body is just a map lookup or a one-time renderer construction, so
  contention is bounded; still worth a comment that `sync.RWMutex` would not help
  here because the build branch needs the write lock anyway.
- The cache grows unbounded over the lifetime of the process if the terminal is
  resized many times to many distinct widths. Realistic widths are bounded
  (a few hundred values at most), so this is fine, but a brief comment in the code
  saying so would prevent a future drive-by "we should LRU this" PR.
- Confirm `applyTheme` calls `InvalidateMarkdownRendererCache()` before the next
  render frame; otherwise the first frame after a theme change will render with the
  old style.

## Recommendation
Land after a short comment on cache-growth bounds and confirmation that
`applyTheme` invalidates before the first post-change render.
