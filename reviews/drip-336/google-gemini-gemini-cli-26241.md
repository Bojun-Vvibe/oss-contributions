# google-gemini/gemini-cli #26241 — fix(cli): resolve tmux scroll issue by using ink's useStdout for terminal size

- **Head SHA reviewed:** `b30b996e73411ff29ec656b2a3b82749ae0db8ed`
- **Size:** +22 / -15 in `packages/cli/src/ui/hooks/useTerminalSize.ts`
- **Verdict:** merge-as-is

## Summary

Fixes #11560: under tmux, the scroll buffer was only filling ~20 % of
the screen because `useTerminalSize` read `process.stdout.columns/rows`
directly, which under tmux's pipe-wrapped stdio doesn't reflect the
attached terminal. The fix routes through ink's `useStdout()` hook
which exposes the resolved TTY stream, with `process.stdout` retained
as a fallback.

## What I checked

- `packages/cli/src/ui/hooks/useTerminalSize.ts:8` — adds `useCallback`
  and `useStdout` imports. Both are correct deps.
- `useTerminalSize.ts:11-21` — `getDimensions` is a `useCallback`
  closing over `stdout`, returning `{ columns: stdout?.columns ||
  process.stdout.columns || 60, rows: stdout?.rows || process.stdout.rows
  || 20 }`. The fallback chain is right: prefer the ink-resolved
  stdout, fall back to `process.stdout`, then to a sane default.
- `useTerminalSize.ts:23` — initial state is `useState(getDimensions)`
  using the function form, so `getDimensions` is only called once on
  mount. Correct, avoids re-computing every render.
- `useTerminalSize.ts:25-37` — effect now (a) calls `updateSize()`
  immediately on mount/whenever `stdout` changes (good — fixes a real
  bug where the initial render under tmux saw default 60×20), (b)
  attaches the resize listener to `target = stdout || process.stdout`,
  not always `process.stdout`. This is the actual functional fix:
  `process.stdout.on('resize', …)` under tmux never fires, but the
  ink-provided stream's resize event does.
- Cleanup (`target.off('resize', updateSize)`) uses the same `target`
  reference captured in the effect closure, so listener removal pairs
  correctly even if `stdout` changes between effect runs.
- Effect deps `[stdout, getDimensions]` — `getDimensions` is itself
  memoized on `[stdout]`, so the effect re-runs exactly when `stdout`
  changes. Stable.

## Concerns

None blocking. Two minor observations:

1. The `||` fallback (`stdout?.columns || …`) treats a column count of
   `0` as "unset" and falls through. In practice ink will never report
   0 columns, but `??` would be technically more correct. Not a
   regression — the previous code did the same.
2. No new test, but the change is a single hook plumbing fix and the
   bug is environment-specific (tmux); a meaningful unit test would
   require mocking ink's `useStdout`. Acceptable for this surface.

## Risk

Very low. The fix preserves the pre-existing fallback path, so
non-tmux environments behave exactly as before. Tmux users see the
real terminal dimensions.

## Recommendation

Merge as-is. Clean, surgical, with a clear bug-to-cause linkage.
