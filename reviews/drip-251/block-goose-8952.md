# block/goose#8952 — fix: make goose2 respect accent color

- **PR:** https://github.com/block/goose/pull/8952
- **Head SHA:** `aea1871b7d4dc2251d7022f9c27e59ed05f19faa`
- **Stats:** +377 / -65 across 13 files (`index.html`, `globals.css`, `ThemeProvider.tsx` + tests, `AppearanceSettings.tsx`, four feature components, two i18n locales, AGENTS.md)

## Context
Goose 2 exposed an accent color picker but the saved color didn't fully propagate through the UI. Several theme aliases still resolved to neutral tokens, so users could choose orange/red/purple but provider cards, model panels, drag-and-drop avatars, and loading spinners stayed visually disconnected from the chosen accent. Sister PR to #8953 (interface density) — both share the same `ThemeProvider.tsx` + `index.html` surface.

## Design
Five coordinated edits:

1. **Pre-mount theme initializer in `index.html:7-86`** — IIFE that runs before React mounts:
   - `normalizeHexColor` accepts `#abc`, `#abcdef`, with or without leading `#`, lowercases output, returns `null` for `"default"` or invalid input
   - `getRelativeLuminance` does the WCAG sRGB → linear conversion (the standard `0.04045 / 12.92 / +0.055/1.055^2.4` formula)
   - `getContrastColor` picks `#000000` or `#ffffff` based on whose contrast ratio is higher
   - Reads `goose-theme` and `goose-accent-color` from `localStorage`, resolves system preference via `matchMedia`, sets `--brand`, `--brand-foreground`, `--color-brand`, `--color-brand-foreground`, plus `accentColor` (CSS native) and `--density-spacing`
   - Bare `try {} catch {}` swallows storage-disabled environments

   This is the right shape — pre-mount initializer means no FOUC on accent. The contrast computation is the load-bearing piece: if the user picks a dark accent (e.g. `#1a1a1a`), the foreground needs to be `#ffffff` for legibility on accent-colored surfaces.

2. **CSS aliases rewired** at `globals.css:+30/-26` — the theme aliases (primary, sidebar, ring, accent, selection, brand) now inherit from the configured accent color rather than resolving to neutral tokens. This is the actual bug fix: previously these aliases were defined in terms of neutral palette tokens, so accent changes never reached the components that consumed `--accent` / `--ring` / etc.

3. **Component swaps** to brand tokens:
   - `AvatarDropZone.tsx:194-196` — drag-over state goes from `border-accent bg-accent/15 ring-accent/20` to `border-brand bg-brand/10 ring-brand/20`, hover from `hover:bg-accent` to `hover:border-brand/50 hover:bg-brand/10`
   - `AgentProviderCard.tsx:400, 423, 454` — spinner `text-accent → text-brand`, active border `border-accent/50 → border-brand/50 bg-brand/10`, status dot `bg-accent → bg-brand`
   - `ModelProviderPanels.tsx`, `ModelProviderRow.tsx`, `ProvidersSettings.tsx` — single-token spinner color swaps
   
   The pattern: `accent` was the alias-that-didn't-update, `brand` is the new chokepoint that ThemeProvider actively writes. Migrating consumers to `brand` is the fix.

4. **AppearanceSettings.tsx changes**:
   - Removes "indigo" from `ACCENT_COLORS` list (5 colors → 4 colors; minor UX change worth a callout)
   - Adds explicit "default" reset button (`resetAccentColor`) and exposes `accentColorPreference` from ThemeProvider so the picker can show the user's *preference* (which may be "default" string-sentinel) vs. the resolved color
   - Layout shift: grid → flex-wrap with `max-w-36 justify-end`, which is a visual change beyond the scope-of-fix (worth its own commit ideally)

5. **ThemeProvider.tsx:+94/-17** — moves the WCAG contrast logic and hex normalization from index.html into the React layer (canonical), exposes `accentColorPreference`, `resetAccentColor`, and applies all accent CSS variables (`--brand`, `--brand-foreground`, `--color-brand`, `--color-brand-foreground`, `accentColor`) in one place during the effect.

## Tests
`ThemeProvider.test.tsx:+130/-2` covers:
- Accent token application (sets all four CSS vars)
- Hex normalization (3-char, 6-char, missing `#`, mixed case)
- Invalid color fallback (e.g. `"blue"` falls back to default)
- Foreground contrast selection (dark accent → white foreground, light accent → black foreground)

The contrast test is the load-bearing one — it locks the WCAG formula. Without it, a future "simplify the contrast computation" refactor could silently regress to "always white" or "always black".

## Risks
- **Duplicated logic between `index.html` IIFE and `ThemeProvider.tsx`** — the same normalization, contrast computation, and CSS-variable application now exist twice. Worth a `// keep in sync with src/shared/theme/contrast.ts` comment in both. (Same risk as #8953.)
- **Removed "indigo" accent without migration** — users who had `goose-accent-color = "#6366f1"` saved will still get that color (because hex values are stored, not names), but the picker UI no longer surfaces it. Worth confirming.
- **`resetAccentColor` exposes a new public API surface** on the ThemeProvider context — make sure no test depends on the old `setAccentColor("default")` shape (if any code path used the string sentinel).
- **`text-accent → text-brand` swaps are visual regressions** for any user who customized via `tui.json`-equivalent overrides expecting `accent` to be the consumer-facing token. Doesn't apply to most users but worth a CHANGELOG note.
- **`AGENTS.md` doc update** says "Tailwind CSS 3 → 4" which is a real version bump, but no `package.json` diff in this PR confirms it. Either the bump is in a separate PR (cross-link please) or this is a docs-only correction (which would be the case if Tailwind 4 already shipped in a prior PR).
- **i18n changes touch only `en` and `es`** for `appearance.accent.colors.default` (1 line each) — confirm 17+ other locales don't need the same key added (would otherwise show the raw key).
- **Layout change in AppearanceSettings** (grid → flex-wrap) is unrelated to the bug fix and would benefit from being a separate commit for cleaner review.

## Suggestions
1. Sync-comment between `index.html` IIFE and `ThemeProvider.tsx` for normalization + contrast logic.
2. Confirm AGENTS.md Tailwind 4 mention matches deployed `package.json` (cross-link the bump PR if separate).
3. Confirm the missing locales for `appearance.accent.colors.default` aren't needed (or add them).
4. Migration note in PR body for indigo removal — users with the saved hex are fine, picker just no longer surfaces it.
5. Optional: split the layout change into a separate commit so the security/correctness fix and the visual refresh are reviewable independently.
6. Negative test: invalid stored accent (e.g. `"not-a-color"`) → default (already covered, good); add `null`/`undefined` storage path explicit test to prove no-crash.

## Verdict
**merge-after-nits** — correct architecture (alias `accent` was the problem, new `brand` chokepoint is the fix, ThemeProvider owns the application), strong WCAG contrast computation with locking test, but duplicated `index.html`/`ThemeProvider` logic, the unrelated layout change, and the i18n/Tailwind-version cross-cuts need cleanup or callouts.

## What I learned
The "alias that doesn't update because the alias resolves to a neutral token" pattern is recurring in CSS-variable theme systems — `--accent: var(--neutral-500)` looks like an accent reference but bypasses the user-configured accent entirely. The fix is to introduce a chokepoint variable (`--brand`) that ThemeProvider actively writes, then migrate consumers to it. The other lesson: WCAG contrast computation belongs *in code* with a locking test (`getRelativeLuminance` + `getContrastColor`), not in a hardcoded `if (luminance > 0.5) "white" else "black"` heuristic, because the threshold-pick differs between WCAG AA, AAA, and APCA standards and you want one canonical formula across mount-time and runtime applications.
