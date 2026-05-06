# QwenLM/qwen-code#3871 — feat(cli): comprehensive i18n — native locale coverage, cleanup, and dynamic translation

- URL: https://github.com/QwenLM/qwen-code/pull/3871
- PR: #3871
- Author: shenyankm (Yan Shen)
- Head SHA: `080c3bf92cfe3412e9096d7057c693fbf1eba789`
- State: OPEN  | +1270 / -456

## Summary

Adds 7 locale files (de, en, fr, ja, pt, ru, zh) under `packages/cli/src/i18n/locales/`, introduces a `mustTranslateKeys.ts` validator with a paired test, refactors `i18n/index.ts` to support dynamic locale loading, and updates `arenaCommand`/`SuggestionsDisplay` for i18n compatibility. New `check-i18n.ts` script (368 lines, replacing the prior version) handles parity checks across locales. Also touches docs and bundle-asset copying. Net +1270/-456 across 21 files including 4 new test files.

## Specific references

- `packages/cli/src/i18n/index.test.ts` (new, 57 lines): test coverage for the refactored loader.
- `packages/cli/src/i18n/index.ts:1-160`: 160-line rewrite of the i18n core (full diff not in window — needs maintainer to spot-check the dynamic-load semantics, especially error handling for missing locale files).
- `packages/cli/src/i18n/locales/{de,en,fr,ja,pt,ru,zh}.js`: 7 new locale files, sizes 19–131 lines (en is smallest at 19 — confirm intentional, since en is usually the source-of-truth and should be the *largest*).
- `packages/cli/src/i18n/mustTranslateKeys.ts` (new, 74 lines) + `mustTranslateKeys.test.ts` (new, 87 lines): validator for keys that must be present in every locale.
- `packages/cli/src/ui/commands/arenaCommand.ts:1-146`: refactored for i18n compatibility (146 lines changed — large enough to deserve a smoke test).
- `packages/cli/src/ui/components/SuggestionsDisplay.tsx:17`: i18n integration with new `SuggestionsDisplay.test.tsx` (68 lines, new).
- `scripts/check-i18n.ts:1-368`: rewritten parity-checker (368 lines, mostly subtractive vs prior version).
- **`docs/users/features/commands.md`**: removes the entire `### 1.7 Session Recap (`/recap`)` section (40+ lines) and the `/recap` row from the commands table. **Source for `/recap` is not in the file list**, so the feature presumably still exists — this is a docs-only deletion.

## Concerns / nits

- **Documentation removal of `/recap` is suspicious**: the PR title and description are about i18n, but `commands.md:27` drops `/recap` from the command table and `:160-208` deletes the entire section explaining `/recap` (auto-trigger on focus, fast-model usage, `general.showSessionRecap` setting). The `/recap` *source* is not in the changed files list, so the feature is still live but undocumented after this lands. Either the deletion is collateral damage from a rebase/merge mistake, or `/recap` is being deprecated separately and the doc removal is intentional but unannounced. **Needs explicit answer from author before merge.**
- `en.js` is 19 lines vs `de.js` (86), `ja.js` (131), `zh.js` (116). En should typically be a superset (or at minimum complete). Confirm whether `en` falls back to source strings in code (and the locale file is just overrides) or whether en is genuinely under-translated.
- Bundle asset script change (`scripts/copy_bundle_assets.js +12`) is needed for new locales but should ship with a corresponding test or smoke check that the published `npm pack` actually includes the locale files.
- 1270/-456 is large for a feature PR; splitting into (a) i18n core refactor, (b) new locale additions, (c) check-i18n script rewrite would have made review tractable. Too late for this PR but a process note.
- No CHANGELOG entry visible.

## Verdict

**needs-discussion (nd)** — load-bearing concern is the silent removal of `/recap` documentation while the feature appears to still exist. Holds for author clarification on intent (documented deprecation, accidental delete, or out-of-scope cleanup) before merge.
