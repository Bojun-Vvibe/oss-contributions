# sst/opencode #24646 — docs(ecosystem): add opencode-honcho plugin

- **PR**: [sst/opencode#24646](https://github.com/sst/opencode/pull/24646)
- **Head SHA**: `1c5c6935`

## Summary

Single-row append to the Plugins section of the ecosystem docs page at
`packages/web/src/content/docs/ecosystem.mdx:55`, listing
`opencode-honcho` (a Honcho-backed long-term-memory plugin from
plastic-labs). One added line, no other changes.

## Specific findings

- `ecosystem.mdx:55` — appended after the existing `opencode-firecrawl` row,
  preserves column alignment (the file uses a fixed-width-ish padding
  convention so rows render aligned in raw view). Padding count is consistent
  with the surrounding rows.
- Link target `https://github.com/plastic-labs/opencode-honcho` resolves to a
  real, public repo authored by the same org that ships Honcho itself
  (plastic-labs/honcho), so this is a first-party plugin from the upstream
  memory product, not a third-party shim. Reduces the risk that the plugin
  becomes orphaned shortly after listing.
- Description text ("Persistent long-term memory across sessions, context
  wipes, and restarts via Honcho") fits the table's
  one-sentence-capability-summary convention and doesn't oversell.

## Nits / what's missing

- The Plugins table is alphabetized loosely-by-add-time, not strictly. A
  future maintainer pass to sort alphabetically would make new entries
  easier to slot in, but that's a docs-hygiene refactor unrelated to this
  PR.
- No CI check verifies that listed plugin URLs are reachable. Stale-link
  rot in this table is a known low-grade problem (look at the existing list
  — at least one of the older rows points at a repo that's been archived).
  Out of scope for a single-row append, but worth a future automation pass.

## Verdict

**merge-as-is** — minimal docs addition, real plugin from a credible upstream,
no behavior changes. Standard ecosystem-table contribution.
