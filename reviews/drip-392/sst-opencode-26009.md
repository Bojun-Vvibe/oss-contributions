# sst/opencode #26009 — fix: make session scrollbar track visible on default theme

- PR: https://github.com/sst/opencode/pull/26009
- Head SHA: `10adf03f448e540f6d0f44b881ccb614a1c85fdd`
- Base: `dev`
- Size: +2 / -2 across 1 file
- Files: `packages/opencode/src/cli/cmd/tui/routes/session/index.tsx`

## Verdict
**merge-as-is**

## Rationale
Two-line theme-token swap that fixes the exact issue described in #25981: with `scrollbar_visible` enabled, the session scrollbar uses `theme.backgroundElement` (#1e1e1e) for its track and `theme.border` (#484848) for its thumb. On the default dark theme both blend into the surrounding panel, so the scrollbar is effectively invisible.

The fix at `packages/opencode/src/cli/cmd/tui/routes/session/index.tsx:1070-1071` swaps to:
- `backgroundColor: theme.background` (#0a0a0a) — track now blends *into the viewport background* rather than the panel chrome, which is the correct visual model: the scrollbar shouldn't be its own UI surface, it should sit on the content surface.
- `foregroundColor: theme.borderActive` (#606060) — gives the thumb meaningful contrast against the new track.

Author cites the same pattern already in use by the sidebar at `sidebar.tsx:40-44`. Reusing an established working precedent in the codebase is the right move; it also means whoever themes this later only has one mental model to maintain.

The change is a pure token swap — no logic, no API surface, no test surface. The author's manual verification (running `bun dev .` with `scrollbar_visible` temporarily forced to true) is appropriate for a TUI styling fix.

## Specific lines
- `packages/opencode/src/cli/cmd/tui/routes/session/index.tsx:1070` — `backgroundColor: theme.background` (was `theme.backgroundElement`).
- `packages/opencode/src/cli/cmd/tui/routes/session/index.tsx:1071` — `foregroundColor: theme.borderActive` (was `theme.border`).

Both align with the sidebar precedent. No nits. Ship it.
