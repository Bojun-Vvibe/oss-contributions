# sst/opencode #24588 — docs(it): fix duplicated word in CLI env var table

- **Repo**: sst/opencode
- **PR**: #24588
- **Author**: SeashoreShi
- **Head SHA**: 7d35c9abbd6fba5ac65b4a34bdea859c8a31c740
- **Base**: dev
- **Size**: +1 / −1 in a single file:
  `packages/web/src/content/docs/it/cli.mdx`.

## What it changes

Single-character-class typo fix in the Italian CLI documentation
table for experimental environment variables. The
`OPENCODE_EXPERIMENTAL_LSP_TY` row had its description start with
the word "Abilita" repeated:

```diff
- `OPENCODE_EXPERIMENTAL_LSP_TY` | boolean | Abilita Abilita TY LSP per i file python
+ `OPENCODE_EXPERIMENTAL_LSP_TY` | boolean | Abilita TY LSP per i file python
```

(at `packages/web/src/content/docs/it/cli.mdx:601`).

## Strengths

- One-line, one-character-class fix to a translated string. No
  semantics change, no schema or code touched, no other rows
  modified. The risk surface is exactly the rendered Italian
  table cell.
- Matches the surrounding column convention. The neighboring rows
  ("Abilita strumento LSP sperimentale", "Abilita funzionalità Exa
  sperimentali", "Abilita markdown sperimentale", "Abilita plan
  mode") all start with a single "Abilita". The fix restores
  parallelism with that pattern.
- The variable name and "boolean" type are unchanged, so any
  external doc tooling that scrapes the table by env-var name
  continues to find the row unchanged.

## Risks / nits

- The same translation pattern is worth grep-checking once across
  the rest of the Italian docs tree
  (`packages/web/src/content/docs/it/`) — duplicate-word translation
  artifacts often come in clusters when an automated translation
  pass runs and the proofreader catches one but not all of them. A
  quick `rg -n 'Abilita Abilita|Disabilita Disabilita'`
  (and the equivalent for other common verbs: "Mostra Mostra",
  "Imposta Imposta", "Configura Configura") under that path would
  either confirm this is the only instance or surface a small
  follow-up batch.
- Trailing-whitespace alignment on the modified row preserves the
  original column padding (three trailing spaces before the closing
  pipe), so the markdown table renders identically. No table
  reflow.
- The English source equivalent in the corresponding `en/cli.mdx`
  is presumably the canonical version; not in this PR's scope, but
  worth confirming the English row reads simply "Enable TY LSP for
  Python files" (i.e., that the duplicate-word artifact is
  translation-only and not present upstream).

## Suggestions

- Consider running a one-off audit across all locale docs under
  `packages/web/src/content/docs/<locale>/` for repeated-word
  artifacts; the fix is mechanical and a single PR can clean up the
  whole class.
- If the docs are generated/refreshed by an automated translation
  pipeline, file a follow-up to add a duplicate-adjacent-word lint
  to that pipeline so this class of regression doesn't recur.

## Verdict

`merge-as-is` — pure docs typo fix in a single translated row. No
risk, matches surrounding style. The audit-for-other-locales
suggestion is a follow-up, not a blocker.
