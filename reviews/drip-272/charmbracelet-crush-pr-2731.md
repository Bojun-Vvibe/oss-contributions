# charmbracelet/crush PR #2731 — feat: add theme support with Charmtone default and Gruvbox Dark

- URL: https://github.com/charmbracelet/crush/pull/2731
- Head SHA: `0444e5038117259c51238f2b717e097d935314ea`
- Supersedes: #2593
- Verdict: **merge-after-nits**

## What it does

Adds theme infrastructure on top of the upstream `quickStyle` machinery and
ships a Gruvbox Dark theme alongside the existing Charmtone default:

- `internal/ui/styles/themes.go` (new file, per PR body) — registry with
  `LoadTheme(name)` (case-insensitive) and `BuiltinThemeNames()`.
- `internal/ui/dialog/theme.go` — command palette entry "Switch Theme"
  with a filterable picker, live preview on cursor movement, Esc revert,
  Enter persist.
- `internal/config/config.go:196-201` — adds `TUIOptions.Theme` JSON field
  with schema metadata (default=charmtone, enum examples).
- `internal/ui/common/common.go:40-72` — new `NewCommon(ws, themeName)`,
  `ThemeNameFromConfig(cfg)`, `LoadThemeStyles(name)` helpers.
- `internal/ui/chat/messages.go:164-180` — adds `cacheClearable` interface
  + `ClearItemCaches([]MessageItem)` so cached message renders are
  invalidated on theme change.
- CLI entrypoints `root.go`, `app.go`, `run.go` swap
  `styles.CharmtonePantera()` for `LoadThemeStyles(ThemeNameFromConfig(...))`.

## Specifics I checked

- `internal/cmd/root.go:122` switches `common.DefaultCommon(ws)` to
  `common.NewCommon(ws, common.ThemeNameFromConfig(ws.Config()))`. Backward
  compatible — empty theme name resolves to default.
- `internal/cmd/run.go:192` and `internal/app/app.go:236-241` mirror the
  same pattern in non-interactive paths. Spinners now respect the theme
  too.
- `internal/ui/attachments/attachments.go:42-46, 99-105` adds
  `SetRendererStyles` / `SetStyles` so attachment chips can be re-styled
  on the fly. Necessary for live preview.
- The cache invalidation interface is small and additive — existing
  message item types just need to implement `clearCache()` to participate.

## What I like

- The "live preview, Esc reverts, Enter confirms and persists" UX is the
  right pattern for a theme picker.
- `Styles.Clone()` (per PR description) for deep copy on preview avoids
  the classic mutate-then-revert bug.
- All write paths to `crush.json` go through the existing config-write
  flow rather than ad-hoc disk I/O.

## Nits / concerns

1. **Two themes is a thin starting set.** Adding Catppuccin / Solarized /
   Tokyo Night in the same PR would prove the registry abstraction
   handles palette diversity (different accent counts, light vs. dark).
   Not blocking, but valuable.
2. **`crush-dev` added to `.gitignore`.** Looks like a dev artifact
   leaked in. Confirm it's intended to be a permanent ignore vs. a stray
   local binary.
3. **`charmtoneOpts()` refactor.** PR body says `CharmtonePantera()` now
   delegates to `charmtoneOpts()` for registry reuse. Verify the public
   `CharmtonePantera()` symbol still returns byte-identical `Styles` so
   external callers (if any) don't see subtle pixel diffs.
4. **Theme names in config are free-form strings.** Consider validating
   on load so `"theme": "guvbox-dark"` (typo) errors loudly with the
   list of valid names rather than silently falling back to default.
5. **PR description includes a generated marker** (`💘 Generated with
   Crush — Claude Opus 4.6 via VertexAI`). Some maintainers prefer that
   stripped from the merge commit body.

## Risk

Low. Default behavior (`theme` unset) resolves to Charmtone, which
matches today's appearance. The cache-invalidation interface is purely
additive. Verdict `merge-after-nits` on (2) and (4).
