# continuedev/continue #12201 — chore(deps): bump dompurify from 3.3.3 to 3.4.1 in /gui

- **PR:** https://github.com/continuedev/continue/pull/12201
- **Head SHA:** `843df6c58f21b750883a69c28f267155bd0213cc`
- **Files changed:** 2 — `gui/package.json` (+1/−1), `gui/package-lock.json` (+6/−41)

## Summary

Dependabot dep bump: `dompurify` ^3.0.6 → ^3.4.1 in the `gui/` workspace. DOMPurify 3.4.x release notes call out (a) a fix for on-handler stripping under permissive `CUSTOM_ELEMENT_HANDLING`, and (b) other XSS hardening for HTML-spec-reserved custom-element names (`font-face`, `color-profile`, etc.). The lockfile shows a net **`-35` lines** because removing `request` and bumping `tar` 7.5.10 → 7.5.13 / `esbuild` 0.17.19 → ^0.25.0 drops several transitive deps.

## Line-level call-outs

- `gui/package.json:38` — `"dompurify": "^3.0.6"` → `"^3.4.1"`. Going from a `^3.0.6` caret to a `^3.4.1` caret is the right minor-version bump for a security release. **No semver risk** because both stay within `3.x`. The actual `dompurify` API surface used by Continue's GUI is small (`DOMPurify.sanitize(html, config)`) and 3.4.1 is API-compatible.
- `gui/package-lock.json:179` — `request` removed entirely. `request` has been deprecated since 2020 and frequently flagged for transitive CVEs (`tough-cookie` prototype pollution, `qs` ReDoS). Removing it is **net-positive** but the PR description doesn't mention it — this is a side-effect of the dompurify resolution, not an intentional cleanup, and any code path that depended on `request` being present in the resolution graph would now break. Grep `gui/` for `from 'request'` / `require('request')` to confirm nothing transitive needs it.
- `gui/package-lock.json:184` — `"tar": "^7.5.10"` → `"^7.5.13"`. Patch-level bump, fine. Worth flagging only because `tar` 7.x has had a stream of subtle stream-handling regressions in patch releases; if `gui/` actually uses `tar` directly (vs. transitive), run the integration tests that exercise extraction.
- `gui/package-lock.json:223` — `"esbuild": "0.17.19"` → `"^0.25.0"`. **This is the largest semver jump in the diff** (8 minor versions), pulled in transitively. esbuild 0.18 → 0.19 → 0.20 → 0.21 → 0.22 → 0.23 → 0.24 → 0.25 each had small behaviour changes around CJS/ESM interop, decorator handling, and JSX automatic runtime. The PR title is "bump dompurify" but the practical risk surface is **the esbuild bump**, not the dompurify bump. If the dev build pipeline relies on esbuild quirks (e.g. the CJS named-export interop relaxation that changed in 0.20), this can break the GUI dev build silently. A maintainer needs to run `cd gui && npm run build` on a clean install before merging.
- The PR title and body emphasize the dompurify upgrade and bury the esbuild bump in the lockfile. Dependabot did the right mechanical thing; the human reviewer needs to call out the *actual* breaking-change risk surface, which is esbuild. Comment on the PR asking dependabot (or the author) to either (a) split this into a dompurify-only PR + a separate esbuild PR, or (b) add a build-success note to the description.

## Verdict

**merge-after-nits**

## Rationale

The titular dompurify bump is safe and pulls in a real XSS fix, so the change is net-good. The hidden risk is the esbuild ^0.17 → ^0.25 jump that came along with the resolution — eight minor versions, several with documented semantics changes. Before merging: (1) confirm `gui/` builds cleanly from scratch (`rm -rf node_modules && npm ci && npm run build`), (2) note the `request` removal in the PR body so future bisects can find it, (3) ideally split the esbuild bump into its own PR. As a security bump this is fine; as a "minor dompurify upgrade" the title is misleading.
