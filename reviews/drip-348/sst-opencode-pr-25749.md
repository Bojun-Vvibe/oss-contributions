# sst/opencode#25749 — docs: fix CLI attach section order

- **Head SHA**: `e87ecc7291d9`
- **Verdict**: `merge-as-is`

## Summary

Pure documentation reorder in `packages/web/src/content/docs/cli.mdx` (+31/-31). Moves the `attach` subsection out from under the `agent` heading (where it was sandwiched between `### attach` and `#### create`) to its proper place after `### agent list` and before `### auth`, restoring alphabetical ordering of top-level CLI commands.

## Findings

- `packages/web/src/content/docs/cli.mdx:5-37` — old location: `### attach` appeared as a sibling of `### agent` but visually nested because `#### create` immediately followed it under the agent block, making the rendered TOC misleading. Removal here is correct.
- `packages/web/src/content/docs/cli.mdx:46-76` (new location) — content is byte-identical to the removed block (heading level `###`, same flag table, same example with `opencode web --port 4096 --hostname 0.0.0.0`). Verified by inspection of the diff hunks; no content drift.
- Alphabetical placement: now sits between `### agent` (with its `list`/`create` subcommands) and `### auth`. That's the correct slot for `attach`.
- No other files touched, no link anchors broken (the heading text and slug `attach` are unchanged).

## Recommendation

Trivial, reversible, and the rendered TOC is strictly better. Ship it.
