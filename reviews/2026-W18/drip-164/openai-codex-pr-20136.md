# openai/codex #20136 — Update Codex login success page UX

- **PR:** https://github.com/openai/codex/pull/20136
- **Head SHA:** `bce0fcab3153d459e6baa6c4ba972cfa05025608`
- **Size:** +192 / −150 (1 file: `codex-rs/login/src/assets/success.html`)

## Summary
Visual refresh of the post-OAuth localhost success page to match the desktop auth UX. Adds proper `prefers-color-scheme` dark mode (via `--color-*` CSS variables and a `@media (prefers-color-scheme: dark)` block), a fixed-position wordmark at top-left, a centered `<main>` content shell with rounded "button" CTAs, a `meta name="color-scheme" content="light dark"` hint, and reformats the existing inline SVG favicon onto multiple lines for readability. Pure HTML/CSS asset change — no JS, no auth-flow logic touched.

## Specific observations

1. **Color tokens at `success.html:14-39`** introduce a clean two-tier palette (`--gray-0` through `--gray-1000`, mapped to semantic `--color-*` aliases) and a dark-mode override block that flips backgrounds, borders, and `--text-secondary` to `rgb(255 255 255 / 65%)`. The semantic-vs-raw split is the right pattern — future re-themes touch the `--color-*` aliases only. Two micro-issues: (a) `--interactive-label-secondary-default: #0d0d0d` is hardcoded in the light-mode `:root` and re-declared in the dark-mode block; consider deriving from `--gray-1000`/`--gray-300` like the others for consistency; (b) `--font-use: "SF Pro", ui-sans-serif, system-ui, -apple-system, ...` falls back gracefully but `"SF Pro"` is not auto-installed on Linux/Windows browsers — Inter or `system-ui` should already cover this case, but it's worth confirming this page actually renders the same on Chrome-Linux as on Safari-macOS (the screenshot in the PR description, if any, would only cover one).

2. **Layout shift from absolute-positioned `.container` to flexbox `<main>`** is a meaningful improvement — the old version used `margin: auto; height: 100%; ... position: relative; background: white` (which depends on `<html>`/`<body>` having `height: 100%` set, brittle), the new version uses `body { min-height: 100vh }` + `main { min-height: 100vh; display: flex; align-items: center; justify-content: center; padding: 56px 24px }` (robust). The `.wordmark` is `position: fixed; top: 18px; left: 20px` — at narrow viewports (<360px on phones) it could overlap the centered content. Since this page is served on `localhost` after OAuth and almost certainly viewed on desktop, this is theoretical, but a `@media (max-width: 480px)` rule that switches the wordmark to `position: static` would cost 4 lines.

3. **Title change "Sign into Codex" → "Signed in to Codex"** at `:5` is a meaningful UX win (past tense reflects success state) and matches the page semantics. Good.

4. **No tests, no functional change, the PR body says "static HTML asset only" + "Tests: not run; static HTML asset only"** — accurate and acceptable. There is, however, a snapshot-test risk: if any login integration test reads `success.html` bytes (e.g. for a "served the right asset" assertion), the +192/-150 churn will require snapshot regeneration. Quick `rg "success.html" codex-rs/` to confirm there are no test fixtures keyed on byte-equality would close the loop.

## Verdict: `merge-after-nits`

The visual refresh is uncontroversial and the dark-mode plumbing is done correctly. The two micro-issues (one tier of duplicated color literal, a narrow-viewport wordmark overlap) are minor enough that this could ship today, but the `rg success.html` smoke check is worth doing to catch any byte-fixture tests before the merge.

## Recommended actions
- **(Quick)** `rg "success\.html" codex-rs/ --type rs` to confirm no test fixtures pin asset bytes.
- **(Nit)** Derive `--interactive-label-secondary-default` from existing gray tokens instead of redeclaring `#0d0d0d` / `var(--gray-300)` per-mode.
- **(Nit)** Add a narrow-viewport media query for the fixed wordmark so it doesn't overlap content on phones (theoretical for this page but cheap insurance).
- **(Optional)** Replace `"SF Pro"` lead with `system-ui` for cross-OS rendering predictability — `system-ui` resolves to SF Pro on macOS already.
