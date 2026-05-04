# sst/opencode#25756 — docs(ecosystem): add @datadog/opencode-plugin

- **URL**: https://github.com/sst/opencode/pull/25756
- **Head SHA**: `0a7f8c2826d1`
- **Diffstat**: +1 / -0
- **Verdict**: `merge-after-nits`

## Summary

Single-row addition to `packages/web/src/content/docs/ecosystem.mdx` listing the third-party `@datadog/opencode-plugin` (Apache-2.0, source `datadog-labs/opencode-plugin`) which ships a preconfigured Datadog MCP server.

## Findings

- `packages/web/src/content/docs/ecosystem.mdx:55` — new row inserted at the bottom of the Plugins table, immediately after `opencode-firecrawl`. The table is *not* alphabetically sorted (existing order is roughly chronological — `opencode-daytona`, `opencode-helicone-session`, `opencode-type-inject`, …, `opencode-firecrawl`), so append-at-bottom is consistent with prior entries (e.g. `opencode-firecrawl`, `opencode-sentry-monitor`). No reordering required.
- Pipe alignment / column padding matches the surrounding rows (verified by the diff context — same column widths as the immediately preceding rows).
- Description text "Query Datadog logs, metrics, traces, dashboards, and more via the Datadog MCP" is concise and parallels the style of `opencode-sentry-monitor` / `opencode-firecrawl`. Minor nit: "and more" is hand-wavy — the PR body explicitly enumerates `monitors` and `incidents`. Consider `…dashboards, monitors, and incidents via the Datadog MCP` for parity with the body.
- Link target `https://github.com/datadog-labs/opencode-plugin` should be spot-checked by the maintainer before merging — `datadog-labs` is a side org, not the canonical `DataDog` org, so reviewer should confirm it's officially blessed (an ecosystem listing implicitly endorses external code).

## Recommendation

Plumbing is fine and follows established convention. Tighten the description to enumerate the actual surfaces, confirm the `datadog-labs` org is officially affiliated, then ship.
