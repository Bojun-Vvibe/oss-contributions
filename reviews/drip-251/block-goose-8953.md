# block/goose#8953 — fix: respect goose2 interface density settings

- **PR:** https://github.com/block/goose/pull/8953
- **Head SHA:** `e8f160306aaf4502ae7e0b9a8cc12b46bb690c24`
- **Stats:** +219 / -17 across 5 files (`ui/goose2/index.html`, `globals.css`, `ThemeProvider.tsx` + tests, `AppearanceSettings.test.tsx` new)

## Context
Goose 2 exposed Settings → Appearance → Interface Density (compact / comfortable / spacious) but the saved preference didn't reliably affect the actual spacing tokens consumed by the UI. The settings UI looked like it worked, but reload/persistence was broken because density changes were applied via inline CSS variable mutation rather than as a CSS-owned attribute.

## Design
Four coordinated edits:

1. **Pre-mount initializer in `index.html:7-15`**:
   ```html
   <script>
     try {
       const density = localStorage.getItem("goose-density");
       if (density === "compact" || density === "spacious") {
         document.documentElement.dataset.density = density;
       }
     } catch (_) {}
   </script>
   ```
   Applies the saved density before React mounts, eliminating the FOUC where the page renders at default spacing for ~1 frame before ThemeProvider hydrates. The strict allowlist (`compact || spacious`) is correct: `comfortable` is the default and doesn't need an attribute, and any other value is treated as missing → default. The bare `try/catch` swallows storage-disabled environments (private mode, restrictive CSP) without breaking page load.

2. **CSS-owned tokens in `globals.css:255-264`**:
   ```css
   [data-density="compact"]  { --density-spacing: 0.75; --spacing: 0.1875rem; }
   [data-density="spacious"] { --density-spacing: 1.25; --spacing: 0.3125rem; }
   ```
   The stylesheet now owns the spacing values keyed off the root `data-density` attribute. This is the right boundary: spacing tokens belong in CSS, not in JS-computed inline styles. ThemeProvider just sets/clears the attribute and the cascade does the rest.

3. **ThemeProvider validation** at `ThemeProvider.tsx:+37/-16`. Persisted theme, accent, and density values are validated before being trusted from `localStorage`. The defensive shape: invalid stored values (e.g. `"sepia"` for theme, `"blue"` for accent, `"medium"` for density) fall back to defaults rather than corrupting the theme state.

4. **Tests** — `ThemeProvider.test.tsx:+122/-1` adds five tests:
   - `falls back to default accent color when storage is invalid` (locks the validation contract for accent)
   - `falls back to default theme when storage is invalid` (same for theme)
   - `reads persisted density` (locks the load-time read)
   - Plus existing-test extensions checking that `--density-spacing` and `--spacing` inline-style mutations no longer happen (the negation that proves the CSS-owned-tokens approach replaced the JS-mutation approach)
   
   `AppearanceSettings.test.tsx:+42` (new) exercises the user-flow: render `<AppearanceSettings/>`, click the Compact radio, assert `localStorage.getItem("goose-density") === "compact"` AND `document.documentElement.dataset.density === "compact"` AND inline style for `--density-spacing` is empty (not set by JS).

## Risks
- **The `index.html` pre-mount script is duplicated logic** with `ThemeProvider`'s mount-time read. If the validation rules drift (e.g. ThemeProvider later allows a fourth density value), the inline script will silently mis-classify. Worth a one-line comment in both places noting they must be kept in sync, or extracting the allowlist to a shared `ALLOWED_DENSITIES` const that both consume (harder for the inline script which can't import).
- **No test for the `index.html` inline script itself** — the test exercises ThemeProvider's read, but not the pre-mount initializer. Hard to test without a proper Playwright/Cypress setup, but a brief comment confirming the inline script was manually verified would help.
- **Strict allowlist `compact || spacious` only** — `comfortable` (default) gets no `data-density`, so the `[data-density="compact"]` and `[data-density="spacious"]` selectors don't apply and the stylesheet's `:root { --density-spacing: 1; }` default fires. That's correct, but worth a comment: "comfortable intentionally has no attribute — relies on root default."
- **Migration path for users on old shape** — if any user had inline-style density via the prior shape, removing the JS mutation now means their state on first load gets re-applied via the new attribute path. Should be a no-op for users since the `localStorage` key is preserved, but worth a sentence in PR body confirming.

## Suggestions
1. Comment in `index.html` pointing at `ThemeProvider.tsx` for the canonical validation logic (and vice versa).
2. Add a sibling smoke test that renders without `localStorage` access (mock to throw) and asserts no crash — proves the `try/catch` actually catches.
3. Consider exporting `ALLOWED_DENSITIES = ["compact", "comfortable", "spacious"] as const` from a shared module, even if the inline script can't import it; gives the TypeScript layer a single source of truth that the inline script can be visually compared against during review.
4. PR body could explicitly note that this is companion to #8952 (accent color), since both PRs touch `index.html` and `ThemeProvider.tsx` and the diff shapes are similar — reviewers benefit from the cross-link.

## Verdict
**merge-after-nits** — correct architecture (CSS-owned tokens via root attribute, JS just sets the attribute, pre-mount initializer eliminates FOUC), strong test coverage on the validation contract and the user-flow, but the duplicated allowlist between `index.html` inline script and `ThemeProvider.tsx` needs a sync-comment and the `try/catch` no-storage path needs an explicit test.

## What I learned
The "settings UI updates an inline style that gets reset on reload" pattern is a recurring theming bug because inline style mutations don't survive React unmount/remount cycles cleanly and don't have a pre-mount path. The right shape is *always* CSS-owned tokens keyed off a root attribute (`[data-density="compact"]`), with JS only responsible for setting the attribute — then the cascade and the persistence layer (CSS itself + the attribute) own the spacing semantics. Plus the pre-mount inline-script pattern in `index.html` is the canonical way to eliminate FOUC for theming preferences, with the cost being a small duplication of validation logic that needs a sync comment.
