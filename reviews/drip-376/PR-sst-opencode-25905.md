# sst/opencode #25905 — feat: make modified files in sidebar clickable to open with OS default app

- Author: JosephITA
- Head SHA: `62ab5177d5ef33b5b7c987a29f11220c5c0f433b`

## Files changed (top)

- `packages/opencode/src/cli/cmd/tui/feature-plugins/sidebar/files.tsx` (+9 / -2)

## Observations

- `packages/opencode/src/cli/cmd/tui/feature-plugins/sidebar/files.tsx:30-34` — `onMouseUp={() => openFile(path.resolve(props.api.state.path.directory, item.file)).catch(() => {})}` swallows all errors silently. If `open()` fails because the file was deleted between display and click, or no default app is registered for the extension, the user gets zero feedback. Consider surfacing a toast / error log via the existing TUI notification surface.
- `:30` — `onMouseUp` fires on any mouse button release while the row is hovered; in a dense sidebar this can trigger on right-click context-menu dismiss too. `onClick` (left-button only) is the more conventional choice and matches user expectation for "open file."
- `:35` — `fg={theme().primary}` change makes every modified-file row use the primary accent color. This is fine for indicating clickability, but the previous `theme().textMuted` was deliberate visual hierarchy (sidebar items are secondary content). Worth confirming with design that the entire list now competing with primary accents is intentional; an underline-on-hover or subtle hover-state-only color shift might be a less aggressive affordance.
- `:3-4` — adding `open` and `path` as direct module imports at the file scope is fine; `open` package is already a transitive dep used elsewhere in opencode for similar OS-handler launches, so no new dep concern.
- No tests added for the click handler. Acceptable for a TUI mouse interaction (hard to unit-test), but a snapshot/render assertion that the row exposes an `onMouseUp` would be nice.

## Verdict

`merge-after-nits`

Useful UX improvement. Two nits before merge: (1) replace `onMouseUp` with `onClick` to avoid right-click-release misfires, and (2) at minimum log the swallowed `openFile` error so users can see why their click did nothing. The color change is a design call rather than a code issue.
