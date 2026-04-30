# sst/opencode PR #25070 — docs(web): fix zh-cn cli experimental table

- PR: https://github.com/sst/opencode/pull/25070
- Head SHA: `7c2b13436d99309be4fd8449f0b6a46e40425719`
- Files touched: 1 (`packages/web/src/content/docs/zh-cn/cli.mdx` +3/-2). Net +1 line.

## Specific citations

- The malformed table row is at `packages/web/src/content/docs/zh-cn/cli.mdx:590` (header separator) and `:601` (the merged `OPENCODE_EXPERIMENTAL_LSP_TY` / `OPENCODE_EXPERIMENTAL_MARKDOWN` row). Pre-PR, the separator carried five extra `| --- |` runs that the markdown table renderer was treating as ghost columns, and two distinct env vars had been collapsed into a single line by an inline `|` continuation, so on the rendered docs page only one of the two entries was visible while the column headers showed phantom slots.
- The fix at `:590` strips the separator back to the canonical 3 columns matching the header on `:589`, and `:601-602` splits the collapsed `OPENCODE_EXPERIMENTAL_MARKDOWN` row back onto its own line so it joins `OPENCODE_EXPERIMENTAL_LSP_TY` as a sibling rather than a tail-fragment of the same row.
- Scope is the zh-CN translation only; the en table at the equivalent path was already well-formed in a prior PR (this PR doesn't touch it). No source / test changes.

## Verdict: merge-as-is

## Rationale

This is a 1-net-line markdown-table-shape fix in a translated docs file with zero behavior surface. The diff is mechanical, easy to verify visually against the en source, and the change-set has no risk: no code paths, no fixtures, no schemas. The two defects pre-PR were both concrete and visible (the ghost-column separator and the merged-row continuation), and the PR addresses both exactly. The only nit is that the same author should sweep the other zh-CN tables in `packages/web/src/content/docs/zh-cn/` for the same `| --- |` separator-bloat pattern — when this kind of defect ships in one translated table it usually shipped in three — but that's strictly out of scope for the headline fix. No regression test is appropriate for a static markdown table.
