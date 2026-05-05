# charmbracelet/crush #2538 — fix: detect Ghostty light backgrounds

- **Head SHA:** `c6dde60d2f00e8ca39d2fa85ed4c0606d62aa45e`
- **Base:** `main`
- **Author:** owldev127 (Jordan)
- **Size:** +52 / −0 across 3 files
  (`internal/ui/model/ui.go`, `internal/ui/styles/styles.go`,
  `internal/ui/styles/styles_test.go`)
- **Verdict:** `merge-after-nits`

## Summary

When Crush starts in a Ghostty (or any modern) terminal that supports
the OSC-11 background-color query, the new `tea.RequestBackgroundColor`
command at `ui/model/ui.go:376` asks for the terminal's background
color. The reply arrives as a `tea.BackgroundColorMsg` and is handled
at `ui.go:491-493` — if `msg.IsDark()` returns false, the UI
re-applies styles tuned for a light background via the new
`DefaultStylesForBackground(false)`. Previously the UI assumed dark
unconditionally, which made Crush hard to read on light-themed
terminals.

## What's right

- **Initial query is dispatched from `Init`, not lazily.**
  `ui.go:376` appends `tea.RequestBackgroundColor` to the initial
  command batch alongside `loadInitialSession()`. So the question
  goes out before the first frame is rendered, and the
  `BackgroundColorMsg` reply arrives early in the event loop —
  meaning the user only sees a brief flash of the dark default
  before the light styles take over (if at all, depending on
  terminal latency). Doing the query lazily on first paint would
  have caused a visible re-theme on every startup.

- **`applyBackgroundTheme()` rebuilds the style-dependent components,
  not just the style struct.** `ui.go:380-388` updates
  `m.com.Styles`, but also re-applies the styles to `m.textarea`
  (`SetStyles(m.com.Styles.TextArea)`), `m.status.help`, and rebuilds
  `m.header = newHeader(m.com)`. Without these, the textarea/help/header
  would have kept the dark-theme styles they were constructed with at
  startup. Catching all four of these is the kind of thing that's
  easy to miss — a quick spot-check shows that's the full set of
  style-consuming subcomponents.

- **`m.sidebarLogo = ""`** (line 387) is a small but correct touch.
  The sidebar logo presumably has hardcoded colors baked into its
  rendered string, so clearing it forces a re-render against the new
  palette on next paint instead of showing a stale dark-themed logo
  on a light background.

- **`updateLayoutAndSize()` is called after the theme swap**
  (`ui.go:493`). Style changes can affect text widths (different
  fonts/weights → different reported widths in some terminals), so
  recomputing the layout once is the safe bet.

- **`DefaultStylesForBackground(bool)` is a clean refactor.**
  `styles.go:502-505` keeps the old `DefaultStyles()` as a thin
  wrapper that calls `DefaultStylesForBackground(true)` — backward
  compatible for any other caller. The new function reuses the
  entire dark-style setup as defaults and only overrides the
  background/foreground/border tokens inside `if !hasDarkBackground
  { ... }` (lines 558-571). This minimizes the light-theme diff and
  makes future palette tuning easy: any change to a non-overridden
  token (e.g., `primary`, `accent`) automatically applies to both
  themes.

- **Light-theme color choices look reasonable** based on charmtone
  conventions: `Salt` (lightest neutral) for base background, `Ash`
  for lighter shade and subtle, `Anchovy` for overlay and border,
  and the foreground inversion uses `Pepper` (darkest) for `fgBase`,
  with `Charcoal` / `Oyster` / `Iron` for the muted ramp. That's the
  expected mirror of the dark theme.

- **Test coverage is adequate for the refactor.**
  `styles_test.go:11-19` asserts that `DefaultStylesForBackground(true)`
  uses `Pepper` for background and `DefaultStylesForBackground(false)`
  uses `Salt` for background and `Pepper` for `FgBase`. Confirms the
  branch fires correctly. Three assertions for a 20-line behavioral
  change is appropriate; testing every token would be brittle.

## Nits / before-merge

1. **No fallback when terminal doesn't reply to OSC-11.** Many older
   terminals (or remote tmux sessions) don't respond to
   `tea.RequestBackgroundColor`, in which case
   `BackgroundColorMsg` simply never arrives and the UI silently
   stays on dark theme. That's actually the right default behavior
   (dark is what the project was tuned for), but worth a debug log so
   users can tell from `crush --debug` why their light terminal isn't
   being detected.

2. **`m.sidebarLogo = ""` will cause a one-frame flicker** when the
   theme swap arrives mid-session (e.g., if Ghostty supports dynamic
   background-color change reporting). Negligible for the common
   "answer arrives during startup" path; more visible if anything
   else triggers a re-detection later. Probably fine, but worth
   noting.

3. **The Ghostty-specific note in the PR title is misleading.** The
   change uses generic `tea.RequestBackgroundColor` / OSC-11, which
   works on any terminal that implements it (iTerm2, WezTerm, modern
   xterm, kitty, etc.). The fix is *triggered* by Ghostty supporting
   the query, but the implementation isn't Ghostty-specific. A
   broader PR title ("detect light terminal backgrounds via OSC-11")
   would set expectations correctly and let other terminal users
   benefit visibly.

4. **Light theme not tested against actual rendered widgets.** Unit
   test only checks color assignment; doesn't catch contrast bugs
   (e.g., a low-contrast `fgMuted` against `bgBase` in light mode).
   A quick screenshot in PR comments showing Crush running on a
   light terminal would do more than any unit test for ergonomic
   verification.

5. **`Background` and `FgBase` field names accessed in test
   (`styles_test.go:16-18`) but not visible in the diff.** Confirms
   the `Styles` struct exposes these as exported fields — fine, but
   reviewers should confirm those names match the actual struct
   defined elsewhere in the file (lines not in diff).

## Risk

- **No effect for users on terminals without OSC-11 support** —
  they continue to see the dark theme they always saw. So the
  blast radius is bounded to "Crush users on light-bg terminals
  that previously looked bad get a usable theme".
- **Mid-session theme swap edge case:** if a user's terminal
  re-reports background color after a theme change, the swap
  triggers `updateLayoutAndSize()` mid-session — likely fine, but
  worth manually exercising once.

## Verdict

`merge-after-nits` — the structural change is correct, the
refactor of `DefaultStyles` into the parameterized version is
clean, and the integration into `Init`/`Update` covers the right
lifecycle hooks. Nit #4 (visual confirmation on a real light
terminal) is the one I'd want before merge; the others are
documentation/observability follow-ups.
