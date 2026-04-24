# PR #2593 — feat: add theme support with Charmtone default and Gruvbox Dark

- URL: https://github.com/charmbracelet/crush/pull/2593
- Author: gurnben
- Head SHA: `aed0a2666c0e551e40e04e6660fe155280adecdd`

## Summary

Introduces a full theming system that decouples UI colors from hardcoded
`charmtone` references. Adds `ThemePalette`/`ThemeColors` (30+ semantic
slots), refactors `DefaultStyles()` into `NewStyles(palette)`, ships two
built-in themes (`charmtone` default, `gruvbox-dark`), wires a "Switch
Theme" command-palette entry with live preview + revert-on-Esc, persists
the choice via a new `options.tui.theme` config field, and propagates the
palette through every CLI surface (`crush run`, `crush session show/list`,
spinner, logos). Default visual appearance is unchanged.

## Specific callouts

- `internal/config/config.go:195-200` — `TUIOptions.Theme` added as a free-form
  string with `enum`-less jsonschema. Two consequences:
  1. The schema documents `example=charmtone,example=gruvbox-dark` but does
     not constrain — typos like `"gruvbox_dark"` will silently fall back via
     `LoadTheme(themeName)`'s zero-error contract. Either tighten to a real
     `enum` (regenerating it whenever a built-in is added) or surface a
     startup warning when `LoadTheme` returned the default because of a
     miss. Silent fallback is the worst UX for a setting users explicitly
     wrote down.
  2. The comment `// Here we can add themes later or any TUI related options`
     was removed cleanly, but the field ordering puts `Theme` between
     `DiffMode` and `Completions` — fine for JSON, but if any code
     constructs `TUIOptions{}` positionally anywhere it'll break (a quick
     `rg "TUIOptions\{[^:}]" --type go` would catch it).
- `internal/cmd/root.go:118-145` — `common.NewCommon(ws, common.ThemeFromConfig(ws.Config()))`
  replaces `common.DefaultCommon(ws)`. The `heartbit` ASCII logo at line 142
  swaps `charmtone.Dolly` for
  `lipgloss.Color(styles.DefaultPalette().Colors.Secondary)`. Note: this
  hardcodes `DefaultPalette()` rather than reading the configured palette,
  which means the splash logo never themes. If that's intentional ("the
  brand splash is always Charmtone"), add a comment; if not, route through
  `ThemeFromConfig`.
- `internal/cmd/update_providers.go:60-72` — Same pattern: uses
  `styles.DefaultPalette()` because "config is not loaded in this command
  context". Acceptable, but the comment is the kind of thing that rots —
  six months from now someone will load config here and forget to swap.
  Consider a `styles.PaletteFromConfigOrDefault(maybeConfig)` helper so the
  fallback is centralised.
- `internal/cmd/session.go:111-138` — Stashes `cfgData` once on
  `sessionServices` and passes it through `outputSessionHuman(ctx, svc.cfg,
  ...)`. Clean. The `hashStyle`/`dateStyle`/`keyStyle`/`valStyle` quartet
  now reads `sty.Blue` / `sty.BlueDark` from the palette — confirms the 30+
  semantic slots include the right granularity for CLI table formatting.
- `internal/app/app.go:232-240` — Defensive nil-walk
  `cfg != nil && cfg.Options != nil && cfg.Options.TUI != nil` before
  reading `cfg.Options.TUI.Theme`. Correct, but `LoadTheme("")` should
  itself tolerate the empty-string case (returning the default), which
  would let this collapse to a one-liner. Worth confirming the
  `_, _ := styles.LoadTheme(themeName)` discard isn't swallowing a real
  error path.
- Theme-picker preview/revert is described in the PR body but the diff
  excerpt I have doesn't show `Styles.Clone()`. Two questions for the
  picker code (not in the visible diff):
  1. Does Esc actually restore *every* cached render (textarea
     completions, attachment chips, status line, spinner) or only the
     palette? The PR mentions a `refreshStyledComponents()` — please
     confirm it runs symmetrically on both apply and revert.
  2. Is the live-preview palette swap atomic, or can a mid-frame
     re-render see a half-applied palette (e.g. background updated but
     foreground not)? If `Styles` is a struct-of-pointers, atomic swap
     of the parent pointer is the safe pattern.
- Tests via `reflect` for `requiredColorFields` struct sync — clever guard
  against forgetting to add a slot to a new built-in. Lock that in with a
  comment explaining the failure mode, otherwise the next person to see
  reflective code in tests will "simplify" it.

## Risks

- 41,781 LOC added across the comprehensive-agent-kernel PR (#2580) is in
  the same release window as this one — large surface, palette propagation
  could collide with new agent UI surfaces that didn't exist when this PR
  was written. Worth a rebase + visual smoke against #2580 before merge.
- Silent fallback for unknown theme names is the single most likely
  user-visible bug. Fix with either schema enum or startup warning.
- Two `DefaultPalette()` hardcodes (heartbit logo, update-providers
  banner) freeze visual elements that users will reasonably expect to
  theme.

## Verdict

**Verdict:** merge-after-nits

The architecture is right and the `NewStyles(palette)` refactor is clean.
Three nits before merge: (1) startup warning when `LoadTheme` falls back to
default, (2) explicit comment on the heartbit/update-providers
`DefaultPalette()` choices, (3) confirm the picker's revert path actually
restores all cached renderers symmetrically.
