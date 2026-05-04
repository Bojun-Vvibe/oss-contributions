# block/goose#8952 — fix: make goose2 respect accent color

- PR ref: `block/goose#8952` (https://github.com/block/goose/pull/8952)
- Head SHA: `aea1871b7d4dc2251d7022f9c27e59ed05f19faa`
- Title: fix: make goose2 respect accent color
- Verdict: **merge-after-nits**

## Review

This PR does three things at once: (1) renames the design-token from `accent` →
`brand` across the goose2 UI, (2) adds an inline pre-React boot script in
`ui/goose2/index.html:7-90` that reads localStorage and applies `--brand`,
`--brand-foreground`, `--color-brand`, `--color-brand-foreground`, and
`--density-spacing` *before* the React tree mounts, and (3) computes a contrast-aware
foreground color via `getRelativeLuminance` + `getContrastColor` so text on the
user-chosen accent stays readable.

The pre-mount FOUC fix at `index.html:54-89` is the right shape — the old approach
deferred theme application until React's `ThemeProvider` ran, which produced a visible
flash of the default accent on cold starts. Computing the contrast color at
`index.html:42-49` using sRGB-linearized WCAG luminance with the +0.05 offset on both
sides and picking the higher of `(L+0.05)/0.05` vs `1.05/(L+0.05)` is the textbook WCAG
2.x algorithm — correct.

The token rename is mechanical and consistent: every `text-accent`, `bg-accent`,
`border-accent`, `ring-accent` reference becomes the `brand` equivalent
(`AvatarDropZone.tsx:194`, `AgentProviderCard.tsx:400, 423, 454`,
`ModelProviderPanels.tsx:36`, `ModelProviderRow.tsx:380, 489, 492`,
`ProvidersSettings.tsx:352`). The new "default" accent button at
`AppearanceSettings.tsx:96-110` with the diagonal black/white gradient is a nice UX
touch and the `accentColorPreference === "default"` check at line 105 is the right
sentinel.

The removal of `indigo` from `ACCENT_COLORS` at `AppearanceSettings.tsx:23` and the
matching i18n key delete at `settings.json` is consistent. Good.

Nits:
1. `index.html:31-49` duplicates contrast logic that almost certainly already lives in
   the React `ThemeProvider`. Fine for the boot path, but worth a comment pointing at
   the canonical implementation so the two stay in sync — the `catch {}` at line 84
   silently swallows divergence today.
2. The `try { ... } catch {}` swallows *all* errors, including `localStorage`
   `SecurityError` in private-browsing contexts and `matchMedia` issues in old
   WebViews. That's defensible (boot script must never crash the page), but a
   `console.warn` inside the catch would help future debugging without changing
   behavior.
3. `getContrastColor` returns `"#000000"` or `"#ffffff"` only — no mid-tone option for
   accents near WCAG inflection (~`#777`). For most palette colors this is fine, but on
   very-mid-luminance accent values the picked text can sit right at the 4.5:1 line.
   Not blocking; a follow-up could pick from `[#000, #fff, theme-fg]`.
4. The `AGENTS.md` change at line 7 says "Tailwind CSS 4" but the rest of the diff
   doesn't show a Tailwind version bump in `package.json`. Verify the doc isn't drifting
   ahead of the actual dependency.
