# sst/opencode PR #25399 — fix(docs): CLI docs for current commands and flags

- URL: https://github.com/sst/opencode/pull/25399
- Head SHA: `afcc62a959f069d1f573ba45964e999e390d008a`
- Closes: #25398
- Verdict: **merge-after-nits**

## What it does

Pure docs PR. Cross-checks the `cli.mdx` reference against the actual CLI
command definitions and (a) adds missing flags/short aliases, (b) wraps every
flag in `<nobr><code>{"--flag"}</code></nobr>` to keep long double-dash flags
from line-wrapping inside narrow Markdown table cells. The change is applied
across every locale file under `packages/web/src/content/docs/<lang>/cli.mdx`.

## Specifics I checked

- The MDX `<nobr>` + `{"--continue"}` JSX-expression trick avoids the MDX
  parser swallowing the leading `--` (which Astro/Starlight has historically
  treated as comment-y in some renderers). Sensible workaround.
- Tables now consistently surface `Short` columns where the underlying yargs
  config defines aliases. Sample spot-check on `ar/cli.mdx` lines 29–43
  matches the `RunCommand` and `AttachCommand` definitions in
  `packages/opencode/src/cli/cmd/run.ts` and `tui/attach.ts`.
- `--keep-config -c` etc. for `uninstall` now have the short column populated.

## Nits

1. The wrapping pattern is `<nobr><code>{"--flag"}</code></nobr>`. That's a
   lot of ceremony per cell and the current diff already shows partial
   conversion in `bs/cli.mdx` (some rows wrapped, some not, e.g. `--prompt`,
   `--model`, `--agent`, `--port`, `--hostname` are still bare backticks at
   lines 36–40 of the new table). Either go all-in or extract a small MDX
   component (e.g. `<Flag name="--prompt" />`) so future edits don't
   silently drift back to wrapping behavior.
2. Touching ~30 locale files in a single PR means non-English docs may now
   contain English-only descriptions for the newly added flags (e.g. the new
   `--variant`, `--thinking`, `--dangerously-skip-permissions` rows are added
   in English in the `ar` file). Consider either (a) leaving the flag listed
   but mark `// TODO: translate` or (b) only touching `en` and letting the
   localization workflow follow up.
3. No test/check that catches the next drift. A tiny script in `packages/web`
   that diffs the documented flag list against the yargs config and fails CI
   would prevent re-doing this PR in three months.

## Risk

Zero runtime risk; docs only. Verdict is `merge-after-nits` purely on (1)
consistency and (2) the localization concern; if the maintainer is fine
shipping English-only newly-added rows in non-English files, this can land
as-is.
