# sst/opencode PR #24614 — `docs(it)`: drop duplicated `Abilita` in IT env-var table

- **PR**: https://github.com/sst/opencode/pull/24614
- **Author**: @SeashoreShi
- **Head SHA**: `7d35c9ab`
- **Size**: +1 / −1
- **Files**: `packages/web/src/content/docs/it/cli.mdx`

## Summary

One-character docs fix in the Italian translation of the experimental-env-vars table. The row for `OPENCODE_EXPERIMENTAL_LSP_TY` had `Abilita Abilita TY LSP per i file python` — two consecutive `Abilita` (it-IT for "Enable") — clearly a copy-paste artifact from the row above. The diff drops one of them.

## Verdict: `merge-as-is`

## Specific references

- `packages/web/src/content/docs/it/cli.mdx:601` — single line change inside the experimental-flags Markdown table:
  ```
  - | `OPENCODE_EXPERIMENTAL_LSP_TY` | boolean | Abilita Abilita TY LSP per i file python   |
  + | `OPENCODE_EXPERIMENTAL_LSP_TY` | boolean | Abilita TY LSP per i file python   |
  ```
  Touches no other locale, no behavior, no schema. The trailing whitespace inside the cell is preserved (matches the column-padding style of the surrounding rows so the rendered table doesn't reflow in the diff).

## Notes

- This is just IT — a quick `rg "Abilita Abilita"` over `packages/web/src/content/docs/` would catch any sibling translations that copy-pasted the same artifact. I'd bet none, but trivially worth a check before merge.
- The other rows in the same table use `Abilita` (transitive) consistently for "Enable X" and `Disabilita` for "Disable X", so the corrected line stays grammatically aligned with its neighbors.

## What I learned

Doc translation tables are a common breeding ground for sed-and-paste artifacts because translators often duplicate a row, change the env-var name, and forget to retype the description column. The smallest version of "make this a habit" is a CI grep over translation files for `\b(\w+)\s+\1\b` (a word followed by itself) — would catch this and similar one-token reduplications across all locales without false positives on legitimate phrases.
