# block/goose PR #8953 — fix: respect goose2 interface density settings

- **PR:** https://github.com/block/goose/pull/8953
- **Author:** kalvinnchau
- **Head SHA:** `a08e986b` (full: `a08e986b7ff844e88fcc97b1f129ee48876ca817`)
- **State:** MERGED
- **Files touched:**
  - `ui/goose2/index.html` (+3 / -9)
  - `ui/goose2/src/features/settings/ui/__tests__/AppearanceSettings.test.tsx` (+42 / -0, new)
  - `ui/goose2/src/shared/styles/globals.css` (+10 / -0)
  - `ui/goose2/src/shared/theme/ThemeProvider.test.tsx` (+120 / -0)
  - `ui/goose2/src/shared/theme/ThemeProvider.tsx` (+26 / -16)

## Verdict

**merge-as-is**

## Specific refs

- `ui/goose2/index.html:62-79` — bootstrap script no longer hardcodes `--density-spacing` per the inline `spacingScale` map. Instead it sets `root.dataset.density = density` only when value is `compact` or `spacious`. This shifts the source of truth from the inline script to CSS variables under `[data-density="..."]` selectors, eliminating the bootstrap/React drift that caused the bug.
- `ui/goose2/src/shared/styles/globals.css:257-266` — the new `[data-density="compact"]` and `[data-density="spacious"]` blocks are the canonical density tokens (`--density-spacing` plus `--spacing`). Comfortable inherits the default `--density-spacing: 1` from the `:root` block at `globals.css:254-256`. That's a clean three-state ladder with `comfortable` as the no-attribute default.
- `ui/goose2/src/features/settings/ui/__tests__/AppearanceSettings.test.tsx:21-39` — the regression test asserts both directions: `localStorage.getItem("goose-density") === "compact"` AND `documentElement.dataset.density === "compact"` AND that the legacy `--density-spacing`/`--spacing` inline styles are *cleared* (empty string). The "cleared" assertions are the important ones — they pin the new contract that density flows via the data attribute, not via inline CSS variables.

## Rationale

Textbook density-settings fix: the bug was that the inline bootstrap script set `--density-spacing` directly while ThemeProvider expected to drive it via CSS, so persisted density values either didn't apply or got clobbered on re-render. The fix swaps to a data-attribute hook (`[data-density]`) which is the right idiom for theme tokens. Validation includes both bootstrap-level (`globals.css`) and React-level (`ThemeProvider.test.tsx` adds 120 lines of coverage) paths. The PR also validates persisted preferences in the theme layer per the body description — preventing invalid `localStorage` values from leaking into theme state. No nits worth raising; ship it.

