---
pr: https://github.com/sst/opencode/pull/25369
head_sha: 25d34b588d719ff8ac10ead72a7d5f962333eec2
author: rbonestell
additions: 1
deletions: 0
files_touched: 1
---

# Scope

Single-line addition to the ecosystem docs page registering the third-party
`constellation-opencode` plugin in the table at
`packages/web/src/content/docs/ecosystem.mdx:55` (head SHA `25d34b5`). Pure
docs change, no code paths touched.

# Specific findings

- `packages/web/src/content/docs/ecosystem.mdx:55` — new row inserted in
  alphabetical neighborhood of existing third-party entries (sentry-monitor,
  firecrawl). Column alignment in the markdown table is preserved
  (whitespace pad on both link and description cells matches surrounding
  rows visually; the rendered table is unaffected either way).
- The linked URL `https://github.com/ShiftinBits/constellation-opencode`
  resolves but the PR body's reference doc URL
  (`https://docs.constellationdev.io/plugins/opencode`) is unverified by the
  diff; reviewers/maintainers should confirm the plugin actually works
  against the current plugin API before merge.
- No `CODEOWNERS`/lint hits expected. PR description's "tested my changes
  locally" checkbox is unchecked but that's acceptable for a one-line table
  insertion.

# Suggested changes

1. Maintainer should do a 30-second sanity check that the linked plugin
   repo has at least one tagged release and a working install path before
   shipping the listing — the ecosystem page is a low-friction supply-chain
   surface and stale/broken links accrete.
2. Consider whether the table needs sort discipline (alphabetical?
   chronological?). Right now insertion order is ad hoc; not blocking on
   this PR but worth a follow-up issue.

# Verdict

`merge-as-is`

# Confidence

High — the entire diff is one markdown table row; risk surface is the
external link's quality, not the change itself.

# Time spent

~4 minutes.
