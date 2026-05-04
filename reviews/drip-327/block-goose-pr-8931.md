# block/goose #8931 — localize hardcoded strings in provider settings UI

- SHA: `ce93a8e2157ed2e4112dbd213f78ddd121ec5e1a`
- State: OPEN, +56/-15 across 7 files

## Summary

Replaces hardcoded English strings in `ModelProviderPanels.tsx` and `ModelProviderRow.tsx` (Edit/Add, Save/Saving/Saved, Connect/Retry, and several error messages) with `t()` calls. Adds the corresponding entries to `en/common.json` (add, connect, saved, saving) and `en/settings.json` under a new `providers.errors.*` namespace, including interpolation strings like `"Enter a value for {{label}}"`. Also adds an exception entry in `check-file-sizes.mjs` raising the limit on `ModelProviderRow.tsx` from 500 to 515 lines, justified by the additional i18n boilerplate.

## Notes

- `ui/goose2/src/features/settings/ui/ModelProviderRow.tsx:69`: `useTranslation` is widened from `"settings"` to `["settings", "common"]`. Correct, since the file now references both namespaces (e.g., `t("common:actions.retry")`).
- `:118` adds `t` to the `useCallback` deps array — required and correct.
- `en/common.json:23-29`: new keys are correctly placed inside `actions`. Note that the existing block already had `retry`, `run`, `save` etc. — `add` and `connect` are inserted out of alphabetical order; minor.
- `en/settings.json:228-...`: new `providers.errors` namespace with `fieldRequired`, `fieldsMissing`, plus existing-shape error strings. Interpolation tokens (`{{label}}`, `{{fields}}`) match the call sites.
- `scripts/check-file-sizes.mjs:33-37`: raising the line limit by 15 to accommodate i18n verbosity is a pragmatic choice. The justification text is honest. A lint-time tradeoff worth accepting once.
- No test changes, but this is essentially mechanical string extraction; risk is low. A snapshot test of the row's error-state rendering would be nice insurance against future regressions.
- Only `en/*.json` is updated. If the project has other locale files, they'll fall back to the English value via i18next default behavior, but it's worth confirming there's no CI gate that requires parity across locales.

## Verdict

`merge-after-nits` — alphabetize the new entries in `common.json` for consistency; confirm whether other locale JSON files need stub entries for the new keys; consider a small render-test for the error-message branches.