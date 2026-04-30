# block/goose#8931 — localize hardcoded strings in provider settings UI

- PR: https://github.com/block/goose/pull/8931
- Head SHA: `ce93a8e215...`
- Author: delkc
- Files: 6 changed (settings panels + EN/ES locale JSON + size-budget exception), ~+45 / −15

## Context

Several user-facing strings in goose2's model-provider settings UI were still hardcoded English literals: button labels (`Edit`, `Add`, `Save`, `Saving...`, `Saved`, `Connect`, `Retry`), and error messages (`Failed to load provider settings`, `Failed to save`, `Failed to remove`, `Failed to complete sign-in`, `Enter a value for ${field.label}`, `Fill in ${missingLabels.join(", ")}`). These survived the earlier i18n sweep because they were inside conditional render branches and async-callback `setError` calls. This PR pulls them all into the existing i18next setup (`common.json` for shared verbs, `settings.json` for domain-specific errors) with EN + ES translations.

## Design

Three buckets of changes:

1. **`ModelProviderPanels.tsx`** — replaces the literal `"Edit"/"Add"` ternary at line 124-128 with `t("common:actions.edit") / t("common:actions.add")`, and the `idleLabel/pendingLabel/successLabel` props on the Save `<AsyncButton>` at line 287-291 with the corresponding `common:actions.{save,saving,saved}` keys.
2. **`ModelProviderRow.tsx`** — six call sites changed:
   - `useTranslation("settings")` widened to `useTranslation(["settings", "common"])` at line 70.
   - Five `setError(...)` calls (lines 113, 184, 229, 244, 304, 321) and one `setSetupError` call (line 184) now route through `t("providers.errors.{loadFailed,signInFailed,fieldRequired,fieldsMissing,saveFailed,removeFailed}")`. The two interpolating ones use the i18next `{{var}}` template syntax: `t("providers.errors.fieldRequired", { label: field.label })` and `t("providers.errors.fieldsMissing", { fields: missingLabels.join(", ") })`.
   - The Connect/Retry button at line 379-381 changed from `setupError ? "Retry" : "Connect"` to `setupError ? t("common:actions.retry") : t("common:actions.connect")`.
   - The `useCallback` dep array at line 119-121 (loadFields) gains `t` as a dep — required because the function now closes over `t`. Correct fix; if you don't include it, the cached callback will keep referencing the old `t` after a locale change and silently render stale strings.
3. **Locale JSONs** — EN + ES `common.json` get the missing `add`, `connect`, `saved`, `saving` keys; EN + ES `settings.json` gain the new `errors` namespace with the six keys. The interpolation templates in ES are translated correctly: `"fieldRequired": "Ingresa un valor para {{label}}"`, `"fieldsMissing": "Completa los campos: {{fields}}"`.

The size-budget exception at `scripts/check-file-sizes.mjs:30-34` is the operational consequence of the i18n sweep: `ModelProviderRow.tsx` ticks past the 500-line ceiling because each error literal becomes a 3-line `t()` call with interpolation + indented closing paren. The justification ("Splitting would create a thin wrapper with no structural benefit") is honest — the file is at 515 lines, the work is mostly mechanical, and a wrapper component just to dodge a budget would be worse than the exception.

## What's good

- **Adding `t` to the `loadFields` `useCallback` deps** is the kind of lint-clean detail that's easy to miss. Without it you get a stale-closure bug where a locale switch doesn't update error messages until the component remounts. The fact that the PR caught it (and the PR-body presumably acknowledged the budget bump as a side-effect) suggests careful authorship.
- **Interpolation via i18next `{{var}}` templates** is the right pattern — preserves variable substitution in target languages where the position of the variable in the sentence differs ("Completa los campos: {{fields}}" puts the colon before the variable, which is standard ES punctuation).
- **`useTranslation(["settings", "common"])` array form** correctly declares both namespaces this component needs, instead of doing two separate `useTranslation` calls. Good i18next idiom.
- **Symmetric coverage across EN and ES** — every new key has both translations. PRs that ship only EN and bolt on ES later are a frequent source of drift; this one ships them together.
- **Honest size-budget exception with named justification** rather than silently raising the global budget or splitting the component into a fake "presentation/container" pair just to dodge the rule. The right governance posture.

## Risks / nits

- **Other locales not updated** — only EN and ES are in the diff. If goose2 ships fr/de/ja/zh/etc. files, those are now in the "missing-key" state and will fall back to the EN string at runtime, which i18next handles silently. That's not a regression (the fallback is the same string the user saw before), but the locale-completeness CI (if any) should flag it. Worth confirming whether other locales exist in `ui/goose2/src/shared/i18n/locales/` and whether the PR should batch them.
- **`t("providers.errors.removeFailed")` → "Failed to remove"** — generic. If the underlying error from `onRemove` carries useful info, the existing `nextError instanceof Error ? nextError.message : ...` pattern preserves it for the Error case, so the localized fallback is only used when the thrown value isn't an Error instance. Good behavior; just confirm the localized strings aren't ever shown in preference to a more useful upstream message.
- **`fieldsMissing: "Fill in {{fields}}"`** — when `missingLabels.length === 1`, the ES translation `"Completa los campos: {{fields}}"` (plural "fields") will read awkwardly for a single missing field. i18next supports plural forms via `count` parameter; not blocking for v1 but worth a TODO.
- **Size-budget exceptions accumulate** — the `EXCEPTIONS` map in `check-file-sizes.mjs` already had entries for Workspace widget tests and Sidebar.tsx (per the diff context). Each exception is individually justified, but the trend is the warning sign — at some point a refactor pass to split the chunkiest panels is owed. Not this PR's job.

## Verdict

**merge-after-nits** — the i18n mechanics are correct, the dep-array fix is the right kind of careful, and the size-budget exception is honest. Before merge:

1. Confirm whether other locales exist in `ui/goose2/src/shared/i18n/locales/` and decide whether to batch them in this PR or land EN/ES first and follow up.
2. Optionally add a `count`-aware plural for `fieldsMissing` if the codebase already uses i18next plural forms elsewhere; otherwise mark as a v2 polish item.

Nothing else is blocking; the rest of the PR is mechanical-but-correct localization work.
