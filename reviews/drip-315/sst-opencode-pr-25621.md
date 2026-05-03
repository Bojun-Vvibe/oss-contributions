# sst/opencode PR #25621 — fix(app): add Chinese general settings translations

- Link: https://github.com/sst/opencode/pull/25621
- Head SHA: `8f5cbab34311b62233c184f056227d900ba08ade`
- Author: kill74
- Size: +24 / −29 (1 file)

## Files changed
- `packages/app/src/i18n/en.ts` — only the English source dictionary; the actual SC/TC translation files referenced in the PR description are not in this diff.

## Reasoning

PR title says "add Chinese general settings translations", but the diff only touches `packages/app/src/i18n/en.ts`. There are no zh-CN / zh-TW files added or modified in the diff that GitHub returns. Either the locale files were dropped during a rebase, or the PR description is misleading. That gap is the main reason this isn't merge-as-is.

What the diff actually does to `en.ts` is reasonable on its own — it reorganizes the dictionary entries and tweaks several copy strings:

- `packages/app/src/i18n/en.ts:715` introduces a new `settings.general.row.shell.title` block that uses `"Shell"` and `"Auto (default)"` / `"Terminal only"` (lowercase 'd', lowercase 'only'):
  ```
  "settings.general.row.shell.title": "Shell",
  ...
  "settings.general.row.shell.autoDefault": "Auto (default)",
  "settings.general.row.shell.terminalOnly": "Terminal only",
  ```
  while the prior version (further down) had `"Terminal Shell"` / `"Auto (Default)"` / `"terminal only"`. The new casing is more consistent, but several sibling sections in the same file (e.g. `"Code Font"`, `"UI Font"`) still use Title Case, so this is a partial style migration.

- The shell description string at `packages/app/src/i18n/en.ts:717` is rewritten as `"Choose the shell used by the terminal. Compatible shells are also used for agent tool calls."` — semantically equivalent to the old "Choose the shell used for your terminal." but no `description` key is paired with the new `shell.title` block? Actually it is — but the old `colorScheme.description` ("Choose whether OpenCode follows the system, light, or dark theme") is replaced with a much weaker `"Choose the color scheme used by the app"`, and `font.title` is changed from `"Code Font"` to `"Font"`, which loses important disambiguation versus `terminalFont` and `uiFont` defined right next to it.

- The `theme.description` change from `"Customise how OpenCode is themed."` to `"Choose the app theme"` and similar US-spelling normalizations (`"Customise"` → `"Customize"`) suggest the PR author also did a localization-of-English pass that the title doesn't mention.

## Verdict

`request-changes`

Reasons:
1. The PR title and description promise Chinese translation files but the diff does not add or modify any zh-CN / zh-TW resource. The maintainer needs the actual translation files re-added before this can land — otherwise the i18n bug `#25604` isn't fixed.
2. Several user-facing English strings are weakened (`"Code Font"` → `"Font"`, `colorScheme.description` loses the explicit system/light/dark wording). These should be reverted unless the author has an explicit reason.
3. The Title-case → sentence-case migration is inconsistent (only some keys); either commit to it across the file or leave the existing strings alone.
