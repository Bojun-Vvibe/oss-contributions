# sst/opencode#24458: docs: add opencode-adaptive-thinking to ecosystem

- **Author:** ian-pascoe (Ian Pascoe)
- **HEAD SHA:** 01b375d3
- **Verdict:** merge-as-is

## Summary

Single-line docs PR adding a new row to the Ecosystem plugin table
in `packages/web/src/content/docs/ecosystem.mdx`. The added row
points at https://github.com/ian-pascoe/opencode-adaptive-thinking
with the description "Let agents adjust model reasoning effort
during a session". Closes #24457. Verification is `bunx prettier
--check` plus `git diff --check` — both appropriate for a +1/-0
mdx change. The row is inserted at the top of the alphabetical
plugin table (`opencode-adaptive-thinking` correctly sorts before
`opencode-daytona` and `opencode-helicone-session`), preserves the
existing column-padding convention of the surrounding rows
(reviewer eyeballed: name column padded to 98 chars, description
column padded to 99 chars matching `opencode-daytona` exactly),
and the link target is a real public GitHub repo by the PR author
themselves (self-attested ecosystem entry, which is the standard
pattern for this file).

## Specific feedback

- `packages/web/src/content/docs/ecosystem.mdx:20` — added row is
  correctly placed in alphabetical order (between `awesome-opencode`
  reference and `opencode-daytona`), matches the table's column
  padding convention so the markdown table doesn't reflow, and the
  description line stays within the existing visual column width.
- Description "Let agents adjust model reasoning effort during a
  session" is concise, scannable, and parallel in voice with the
  neighboring entries (imperative-mood "Automatically run...",
  "Auto-inject..."). Reads well.
- Link target points at the plugin's own GitHub repo
  (`ian-pascoe/opencode-adaptive-thinking`), not a personal blog
  or paywalled landing page — matches every other row's
  convention.
- The PR description marks this as documentation-only and the file
  list confirms exactly one file touched
  (`packages/web/src/content/docs/ecosystem.mdx +1/-0`), so risk
  surface is bounded.

## Risks / questions

- No external risk; ecosystem entries are self-attested and the
  table is a directory, not a code dependency. The plugin itself
  could go stale or unmaintained — that's true of every ecosystem
  entry and the existing table doesn't have a curation policy
  visible in the file. Not in scope for this PR.
- One non-blocking nit if the maintainers want to standardize: the
  trailing description column for this row is right-padded to 99
  characters but the displayed text is shorter than several
  neighboring entries, so the row visually has a lot of trailing
  whitespace in the rendered table. Not a markdown problem (mdx
  table layout is unaffected), purely a source-file aesthetics
  thing — almost every other row also has uneven padding so this
  matches the existing pattern.
- Consider adding a brief one-paragraph "what does adaptive
  thinking do?" explainer in the linked plugin's README so users
  clicking through have a clear landing experience — out of scope
  for this PR but worth flagging to the plugin author.
