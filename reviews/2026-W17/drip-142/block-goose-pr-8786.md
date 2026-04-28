# block/goose#8786 — Goose2 adaptive theme selection (system + named themes + accent overrides)

- **PR**: https://github.com/block/goose/pull/8786
- **Author**: @tulsi-builder
- **Head SHA**: `b3bc1a3`
- **Base**: goose2 main branch
- **State**: OPEN
- **Scope**: large — 19 files (+2066/-410). Bulk in `ThemeProvider.tsx` (rewrite), new `adaptive-theme.ts`, `theme-loader.ts`, `globals.css` (~330 lines), 4 new test files, several `ai-elements/*` color refactors, e2e smoke spec.

## Summary

Replaces the existing 3-option dark/light/system ToggleGroup in Goose2's `AppearanceSettings` with a searchable theme picker over a loadable theme catalog (`SYNTAX_THEMES` + `ACCENT_COLORS` from a new `theme-loader.ts`), gated by a "System Default" entry that defers to `prefers-color-scheme`. Theme application is reworked to a derived-vars cache in a rewritten `ThemeProvider` that (a) falls back to the system theme when no explicit theme is set, (b) applies accent overrides only for explicit named themes (not for system mode), and (c) bridges the legacy `--theme-*` CSS vars to shadcn `hsl(var(--…))` tokens via a `color-mix(...)` adaptation layer in `globals.css`.

## Diff anchors

- `ui/goose2/src/features/settings/ui/AppearanceSettings.tsx` — replaces the ToggleGroup with a searchable list + System Default sentinel. `ACCENT_COLORS` and `SYNTAX_THEMES` are imported from the new theme-loader; `useTheme()` is the single source of truth for the active theme. Removes the old hard-coded enum.
- `ui/goose2/src/shared/theme/ThemeProvider.tsx` — full rewrite. New behaviors:
  - **System fallback when no theme set**: subscribes to `matchMedia("(prefers-color-scheme: dark)")` and re-derives vars on change. Right shape — uses `addEventListener("change", …)` (not the deprecated `addListener`) and tears down on unmount.
  - **Derived-vars cache**: theme-token computation is memoized per `(theme, accent, mode)` tuple. Avoids the O(themes × props) recompute on every render; the previous code recomputed on every `useTheme()` call.
  - **Accent overrides only for explicit themes**: when in System Default mode, accent picks are intentionally suppressed (the rationale: system mode should look "native," and applying an arbitrary accent on top breaks that). This is a deliberate UX choice and worth a doc comment naming it explicitly so the next person doesn't "fix" it.
- `ui/goose2/src/shared/styles/globals.css` (around the new section) — adaptive var → shadcn token bridge. Uses `color-mix(in oklch, var(--theme-bg) 92%, transparent)` to derive shadcn surface tones from the legacy theme vars. The `oklch` color space (not `srgb`) is the right choice — it preserves perceptual lightness across hue shifts. Confirm browser support floor matches the rest of Goose2's CSS (Chrome 111+ / Safari 16.4+ / FF 113+); if you're targeting older Electron, this needs a `@supports` fallback.
- `ui/goose2/src/shared/theme/adaptive-theme.ts` — new helper that takes a theme descriptor and emits the CSS variable map. Pure function, easy to test, and the test file at `:1958-2014` covers light/dark token derivation symmetry.
- `ui/goose2/src/shared/theme/theme-loader.ts` — new loader. Exports `SYNTAX_THEMES`, `ACCENT_COLORS`. Test at `:2271-2328` covers the catalog shape.
- `ui/goose2/src/shared/ui/ai-elements/{commit,package-info,schema-display,test-results,tool}.tsx` — all six ai-element components migrate from hard-coded color literals (e.g., `text-blue-500`) to theme-aware tokens (`text-[hsl(var(--accent))]` or shadcn token utilities). This is the necessary downstream cleanup so the new accent system actually shows up in those surfaces. Without this PR's bulk rewrite of these, the theme picker would only reskin chrome and not the actual content.
- `ui/goose2/src/shared/ui/sonner.tsx` — toast palette wired to the new tokens.
- `ui/goose2/src/shared/i18n/locales/{en,es}/settings.json` — adds the new strings for theme picker labels. EN + ES coverage; other locales not updated in this PR (worth flagging — most other Goose2 i18n PRs ship all locales together).
- `ui/goose2/tests/e2e/smoke.spec.ts` — new e2e smoke that toggles theme and asserts CSS-var application. Useful regression net for "we ship a theme picker that doesn't actually apply the theme."

## What I'd push back on

1. **Locale coverage incomplete** — only `en` + `es` got the new strings; the rest of the goose2 locale set will fall back to keys. Either ship all locales (with auto-translation if needed) or add an explicit "i18n: pending other locales" note + tracking issue.
2. **`oklch` + `color-mix` browser floor** — confirm against the project's Electron version. Add an `@supports (color: color-mix(in oklch, red, blue)) { … }` block (or `@supports not` fallback) for older Chromium. Otherwise pre-Chrome-111 environments silently get unset vars.
3. **System mode + accent suppression deserves a code comment** — the rationale isn't obvious to a reader. A two-line `// In System mode, accent overrides are intentionally suppressed so the OS theme looks native.` near the relevant branch would prevent regression by future "consistency" patches.
4. **No keyboard-nav test for the searchable picker** — it's a list with a search box. Confirm focus management (arrow keys move selection, Enter applies, Esc closes) and add a single `@testing-library` test. Otherwise the next refactor of the underlying `Combobox` / `Listbox` could silently break a11y.
5. **Snapshot risk in `ThemeProvider.test.tsx`** — confirm tests don't pin transient `--theme-*` color values that change with the catalog. Pin shape, not specific hex.

## Verdict

**merge-after-nits** — the architecture is right (cache derived vars, fall back to system when unset, suppress accent in system mode, bridge legacy vars → shadcn tokens via `oklch` color-mix), the test coverage is broad (4 new test files plus an e2e), and the downstream `ai-elements/*` migration means the theme picker actually does something visible. Block on i18n completeness and the `color-mix`/`oklch` browser-floor `@supports` guard before merging.

Repo coverage: block/goose (Goose2 UI theme system rewrite — adaptive theme provider, system-fallback, accent override semantics, downstream component palette migration, e2e smoke).
