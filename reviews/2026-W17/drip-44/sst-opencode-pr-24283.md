# sst/opencode PR #24283 — docs: add opencode-provider-alias to ecosystem

- **URL:** https://github.com/sst/opencode/pull/24283
- **Author:** @baranwang
- **State:** OPEN (base `dev`)
- **Head SHA:** `4d56599`
- **Size:** +1 / -0 (single-line table insert)
- **File:** `packages/web/src/content/docs/ecosystem.mdx`

## Summary of change

Adds one row to the third-party ecosystem table linking
`baranwang/opencode-provider-alias` — a tool that lets users alias
provider entries and pull model metadata from `models.dev`. Insertion
is at table position immediately after `opencode-firecrawl`.

## Findings against the diff

- The new row matches the surrounding table format (three pipe-separated
  columns, double-spaced for column alignment). Markdown table column
  alignment uses extra spaces — the trailing whitespace on the new line
  is consistent with the four neighboring rows, so no rendering jitter.
- Description is concise and factual ("Alias and curate OpenCode
  providers with model metadata from models.dev"). It avoids
  promotional adjectives, matching the table's tone.
- Link target (`https://github.com/baranwang/opencode-provider-alias`)
  resolves and is owned by the same author, which is the expected
  pattern for third-party ecosystem entries — the author is asking for
  inclusion of their own project rather than someone else's.
- No alphabetical or category ordering is enforced in this table
  (insertion at the end of the current list is the convention).

## Verdict

**merge-as-is**

This is a single-line ecosystem-table addition with a working link, a
plausible scope description, and consistent formatting. There is
nothing to change. The only thing maintainers might want to confirm is
that the linked project actually exists at the time of merge (cheap
to verify), and that they're comfortable with the author self-listing
— but both of those are policy questions, not diff-quality issues.

## What I learned

Ecosystem-table PRs are the lowest-risk surface in any docs site: the
review cost should match the change cost. A ~30-second review (open
the link, sanity-check the description, confirm formatting) is the
right shape. The temptation to over-engineer review process here
(require a category column, sort alphabetically, demand a CODEOWNERS
ack) is the same anti-pattern as adding required CI gates to a
README typo fix — it inflates the cost of trivially-correct
contributions and discourages the long tail of small ecosystem
discoverability work.
