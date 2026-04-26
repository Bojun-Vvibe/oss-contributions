# charmbracelet/crush PR #2538 — fix: detect Ghostty light backgrounds

- **PR:** https://github.com/charmbracelet/crush/pull/2538
- **Author:** owldev127
- **Head SHA:** `c6dde60d2f00e8ca39d2fa85ed4c0606d62aa45e`
- **Files:** 3 (+52 / -0)
- **Verdict:** `merge-after-nits`

## What it does

Fixes #2513. In Ghostty (and any terminal that responds to OSC 11
background-color queries), Crush always rendered the TUI with the dark
palette regardless of the actual terminal background. Result: black-on-dark
text on a light terminal background, unreadable.

The fix:
1. On `UI.Init`, also dispatch `tea.RequestBackgroundColor` so Bubble Tea
   sends an OSC 11 query and reports the result back via
   `tea.BackgroundColorMsg`.
2. In `UI.Update`, handle `tea.BackgroundColorMsg`: build a new style set
   via `styles.DefaultStylesForBackground(msg.IsDark())`, swap it into
   `m.com.Styles`, refresh dependent component styles (textarea, status
   help), and rebuild the header.
3. In `styles.go`, factor `DefaultStyles()` into a thin wrapper over a
   new `DefaultStylesForBackground(hasDarkBackground bool)`. The light
   branch swaps `bgBase`, `bgBaseLighter`, `bgSubtle`, `bgOverlay`,
   `fgBase`, `fgMuted`, `fgHalfMuted`, `fgSubtle`, and `border` to
   light-appropriate `charmtone` colors.

## Specific reads

- `internal/ui/model/ui.go:376` — `cmds = append(cmds,
  tea.RequestBackgroundColor)`. Right place — at the end of `Init`,
  after `loadInitialSession`. The request runs concurrently with the
  rest of init.
- `internal/ui/model/ui.go:380-388` — `applyBackgroundTheme(bool)`.
  Five-line method: rebuild styles, point `*m.com.Styles` at them,
  refresh `textarea` and `status.help` styles, rebuild the header,
  clear `sidebarLogo`. The `*m.com.Styles = sty` is a struct
  reassignment through a pointer — every component already holding
  `m.com.Styles` (by value or pointer) sees the new values, **except
  components that copied a snapshot at Init time**. The textarea and
  status.help refresh handles two of those; if any other component
  cached its own copy of styles, it'll keep the old palette.
- `internal/ui/model/ui.go:491-494` — `case
  tea.BackgroundColorMsg:` calls `applyBackgroundTheme(msg.IsDark())`
  then `m.updateLayoutAndSize()`. The latter is needed because some
  styles influence layout dimensions (border thickness, padding).
  Good catch — without that call you'd see the right colors with the
  wrong layout for one frame.
- `internal/ui/styles/styles.go:501-504` — `DefaultStyles()` keeps
  the old signature and now just delegates to
  `DefaultStylesForBackground(true)`. Backward compatible — any
  call-site that doesn't know about backgrounds still gets dark.
- `internal/ui/styles/styles.go:559-573` — the light-mode block.
  Nine color reassignments, all to `charmtone.*` constants. **The
  semantic mapping** (e.g. `bgBase = charmtone.Salt`,
  `fgBase = charmtone.Pepper`) reads correctly: dark bg → light bg,
  light fg → dark fg, with intermediate shades swapped to
  appropriate counterparts. **But** the test (`styles_test.go`) only
  checks 3 of the 9 reassignments. A proper test would check all 9.
- `internal/ui/styles/styles_test.go:1-19` — new test file. Three
  assertions: `dark.Background == Pepper`, `light.Background == Salt`,
  `light.FgBase == Pepper`. Catches gross regressions but doesn't
  verify e.g. `bgSubtle`, `fgMuted`, `border`. Easy to extend.

## Risk surface

**Low.** UI-only, no protocol or persistence change. Three watch-outs:

1. **Terminals that don't respond to OSC 11.** If a terminal swallows
   the query, `tea.BackgroundColorMsg` never arrives and the UI stays
   on dark default. That's the existing behavior, so no regression —
   but worth a note in the docs that light-bg detection requires OSC 11
   support.
2. **Mid-session background change.** If the user toggles their
   terminal between light and dark mid-session (some users do), the
   query is only sent at Init. Bubble Tea may re-emit
   `BackgroundColorMsg` on focus events; if not, the UI won't follow.
   Out of scope for this fix but worth a follow-up issue.
3. **Race with `loadInitialSession`.** If the initial session loads
   *and* repaints before `BackgroundColorMsg` arrives, users see one
   frame of dark-on-light then it corrects. Probably imperceptible,
   but on slow terminals (high-latency SSH) it could flash. Not a
   blocker.

## Suggestions before merge

- Extend `styles_test.go` to assert all 9 light-mode color swaps.
  Three assertions is too few for a 9-variable change — easy to
  silently regress one.
- Consider re-querying background on `tea.FocusMsg` so mid-session
  light/dark toggles are picked up. Could be a follow-up.
- Document the OSC 11 dependency in the readme or release notes,
  alongside the list of "compatible terminals" if Crush maintains
  one.

Verdict: merge-after-nits — clean, minimal fix that solves a
real-world unreadability bug. The test coverage of the new code path
is thin but the behavior is straightforward enough that the existing
3-assertion test plus the screenshot in the PR is reasonable. Add a
follow-up issue for mid-session toggle handling.

## What I learned

OSC 11 (`\e]11;?\a`) is the canonical way to ask "what's the
terminal background color?" but support is uneven. Bubble Tea
abstracts it via `tea.RequestBackgroundColor` /
`tea.BackgroundColorMsg`. The right pattern for a TUI app is:
- Request once at Init (don't assume dark).
- Have a `DefaultStylesForBackground(bool)` so the styling logic is
  centralized.
- Rebuild dependent component styles when the background message
  arrives, including layout recomputation.
- For the polished version: re-query on focus events so mid-session
  toggles work.

Most TUI apps skip step 1 entirely and just default to dark — which
is exactly the source of bug #2513 in Crush.
