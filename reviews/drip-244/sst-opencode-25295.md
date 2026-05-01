# PR #25295 — fix(tui): avoid opaque flash on startup with transparent custom themes

- Repo: sst/opencode
- Head: `fd8788f049598fa10d2321f4d20a5dbed3466344`
- URL: https://github.com/sst/opencode/pull/25295
- Verdict: **merge-as-is**

## What lands

Closes #23573. Two compounding root causes correctly identified and fixed
together:

1. **Renderer default backdrop**: `createCliRenderer` was called without
   `backgroundColor`, so opentui defaulted to `RGBA(0.1, 0.1, 0.1, 0.7)` — a
   semi-opaque dark layer visible until something forces a full repaint. Fix at
   `cli/cmd/tui/app.tsx:79-85` sets `backgroundColor: RGBA.fromInts(0, 0, 0, 0)`
   in the renderer config.

2. **Theme fallback paints opaque**: `theme.tsx`'s `values()` memo falls back
   to bundled opencode-default (opaque) while a user-installed custom theme is
   still being loaded async from `~/.config/opencode/themes/`. Both the
   `renderer.setBackgroundColor` `createEffect` and the root `<box>`'s
   `backgroundColor={theme.background}` consume that fallback, painting opaque
   cells the later transparent assignment doesn't visually clear. Fix:

   - `context/theme.tsx:431-436` short-circuits the renderer-background
     `createEffect` when `store.themes[store.active]` is undefined.
   - `context/theme.tsx:485-491` exposes new `isActiveResolved()` returning
     `store.themes[store.active] !== undefined`.
   - `app.tsx:884-888` gates the root box's `backgroundColor` on
     `themeState.isActiveResolved() ? theme.background : RGBA.fromInts(0,0,0,0)`.

Default-bundled themes (`opencode`, `monokai`, etc.) are present in the store
from the first synchronous render so they paint their own opaque background
with zero flicker — the gate only activates for the async custom-theme path,
which is exactly the surface area of the bug.

## Why merge-as-is

- The `/theme`-toggle workaround in the original report is correctly identified
  as "dialog dismiss forces full re-render" rather than fixing the underlying
  bug, and the fix is at the *source* of the opaque paint, not at the
  workaround layer.
- Both halves of the fix are necessary: just transparent renderer config still
  leaves the root `<box>` painting opaque cells; just the `<box>` gate still
  has the renderer's default backdrop showing through.
- Inline comments at both call sites accurately explain *why* the gate exists
  rather than what it does — exactly the right comment shape for a future
  reader who'd otherwise be tempted to "simplify" the gate away.
- Tests exist (`theme-store.test.ts` 5/5 pass per PR body) and the change is
  scoped to two files with 22 additions / 2 deletions.

Solid, surgical fix. Ship.