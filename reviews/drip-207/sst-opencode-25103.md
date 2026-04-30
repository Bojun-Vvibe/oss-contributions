# sst/opencode #25103 — tui: themeable dialog and sidebar overlay backgrounds

- Head SHA: `92aba5954566d8da2e1d8ee8ddf7e6d2b7d3f75b`
- Files: `packages/opencode/src/cli/cmd/tui/context/theme.tsx`, `packages/opencode/src/cli/cmd/tui/routes/session/index.tsx`, `packages/opencode/src/cli/cmd/tui/ui/dialog.tsx`, `packages/opencode/test/cli/tui/theme-store.test.ts`, `packages/opencode/test/fixture/tui-plugin.ts`, `packages/plugin/src/tui.ts`
- Size: +43 / -5

## What it does

Closes #25102. Two overlay colors that were previously hardcoded as
`RGBA.fromInts(0, 0, 0, 150)` (the dialog backdrop in
`packages/opencode/src/cli/cmd/tui/ui/dialog.tsx:47`) and
`RGBA.fromInts(0, 0, 0, 70)` (the sidebar overlay at
`packages/opencode/src/cli/cmd/tui/routes/session/index.tsx:1224`) are
now exposed as themeable fields `backgroundDialogOverlay` and
`backgroundSidebarOverlay` on `ThemeJson.theme`.

The wiring follows the same optional-with-fallback pattern as the
existing `backgroundMenu` / `selectedListItemText` / `thinkingOpacity`
fields:

- `theme.tsx:81-87` extends the `Omit` set so the two new keys can be
  optional in the JSON schema.
- `theme.tsx:227` excludes them from the bulk `Object.fromEntries`
  resolution loop (otherwise they'd be processed twice).
- `theme.tsx:250-263` resolves them with explicit RGBA fallbacks
  matching the pre-PR hardcoded values (150 alpha for dialog, 70 for
  sidebar) when the theme JSON doesn't supply them.
- `theme.tsx:594-595` adds `transparent` defaults to `generateSystem`,
  the function that generates a theme from terminal-detected colors —
  meaning auto-detected themes get fully transparent overlays rather
  than the hardcoded black tint, which matches the "let the terminal's
  own background show through" behavior auto-detect implies.
- The two consumers (`dialog.tsx:47`, `session/index.tsx:1224`) drop the
  literal RGBA call and read `theme.backgroundDialogOverlay` /
  `theme.backgroundSidebarOverlay` instead.
- Plugin API (`packages/plugin/src/tui.ts:206-207`) and the test
  fixture (`test/fixture/tui-plugin.ts:41-42`) are widened with the new
  readonly fields so plugins can read them.

Two tests at `theme-store.test.ts:53-67` cover both the default-fallback
path (alpha matches the historical 150/255 and 70/255 ratios) and the
custom-value path (a hex override yields the expected RGBA).

## What works

The pattern mirrors the existing `backgroundMenu` precedent exactly,
which is the right call — adding a sixth bespoke fallback shape would
have been worse than adding a sixth instance of the established one.
The defaults preserve existing visual behavior bit-for-bit for any
theme that doesn't opt in, so this is a strictly additive change for
existing users.

The `generateSystem` decision (use `transparent` rather than the
hardcoded 150/70 black) is a deliberate behavioral nudge that's correct
for the auto-detected case: a terminal-themed user is more likely to
want their terminal background showing through than to want a black
wash. It's worth flagging in the PR description that this *is* a
visible change for terminal-auto themes, not just an inert refactor.

Tests cover both branches of the new code (default and override) and
use `expect(...).toBeCloseTo(150 / 255, 2)` rather than asserting the
exact float, which avoids the rounding-precision flake that
straight-equality on RGBA components tends to produce.

## Concerns

1. **`generateSystem` behavior change is silent.** Switching the
   auto-detected default from `RGBA.fromInts(0, 0, 0, 150)` to
   `transparent` is the correct UX choice but is not called out in the
   PR description, which only mentions "fallback to semi-transparent
   black" — that fallback only applies to the JSON-loaded path at
   `theme.tsx:252-263`. Users on terminal-auto themes will see their
   dialogs lose the black wash overnight. Worth a one-liner in the PR
   body or a CHANGELOG note.
2. **No test for `generateSystem` defaults.** The two new tests cover
   `DEFAULT_THEMES.opencode` (which is JSON-loaded and hits the
   150/70 fallback) but not the `generateSystem` path, which has the
   *different* `transparent` default. A third test that calls
   `generateSystem(...)` and asserts both overlay alphas are 0 would
   pin down the behavioral change so a future refactor doesn't
   regress it back to black.
3. **`Omit` list is now five entries deep.** Each new optional theme
   field requires three coordinated edits (the `Omit`, the resolver
   filter, and the explicit resolution block). This isn't this PR's
   problem to fix, but the `Omit` line at `theme.tsx:81` is now
   approaching the point where a `const OPTIONAL_THEME_KEYS = [...] as
   const` constant would prevent the next addition from forgetting one
   of the three edit sites — particularly the resolver `filter`, where
   omitting a key silently double-resolves it.
4. **Schema/type duplication.** `TuiThemeCurrent` in
   `packages/plugin/src/tui.ts:206-207` and the `ThemeJson` type in
   `theme.tsx` both have to learn about the new keys independently.
   They're related types but not derived from each other, so they can
   drift. Not blocking, but a `type ThemeColor = keyof TuiThemeCurrent`
   shared between the two would be a worthwhile cleanup later.

Verdict: merge-after-nits

## What I learned

When a UI codebase has an "optional with fallback" pattern repeated
five times, the right answer is usually to follow the pattern a sixth
time rather than to refactor it — refactors mid-PR balloon scope and
make review harder. But each repetition shifts a little more weight
onto the *next* contributor to remember all the coordinated edit
sites; eventually one of them silently forgets the resolver-filter
edit and the field gets double-resolved (here: into a string by the
generic loop, then overwritten with an RGBA by the explicit block,
which only works because the explicit block runs second). The cost
isn't paid by this PR — it's paid by the PR after the next one.
