---
pr: 24397
repo: sst/opencode
sha: 2d7806ee039e4ff63f23bc2e23c9987b9d79273c
verdict: merge-as-is
date: 2026-04-26
---

# sst/opencode #24397 — docs: add toon-config-plugin to ecosystem plugins

- **Author**: mmynsted
- **Head SHA**: 2d7806ee039e4ff63f23bc2e23c9987b9d79273c
- **Link**: https://github.com/sst/opencode/pull/24397
- **Size**: 1-line table addition in `packages/web/src/content/docs/ecosystem.mdx`.

## Scope

Adds `opencode-toon-config-plugin` (lets users put agent direction in `AGENTS.toon` files in TOON format instead of `AGENTS.md`) to the published ecosystem plugins table.

## Specific findings

- `packages/web/src/content/docs/ecosystem.mdx:55` — new row added to the existing alphabetically-arranged-ish ecosystem table linking to https://github.com/mmynsted/opencode-toon-config-plugin. Row formatting matches existing entries (column widths, trailing pipe, no markdown link inside the description column).
- The description "Use AGENTS.toon, TOON based AGENTS file when possible over AGENTS.md for OpenCode agent direction." has a slight grammatical awkwardness ("when possible over") but is clear enough — not worth blocking.
- Insert position is below `opencode-firecrawl`; the table is not strictly alphabetical anyway, so position is fine.

## Risk

Zero. Pure docs.

## Verdict

**merge-as-is** — community ecosystem additions like this should land friction-free.
