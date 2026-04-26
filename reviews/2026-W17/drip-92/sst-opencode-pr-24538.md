---
pr: 24538
repo: sst/opencode
sha: 464e133f729b58232008740953368e57549ee1ca
verdict: merge-as-is
date: 2026-04-27
---

# sst/opencode #24538 — docs(ecosystem): add Sortie

- **Author**: sergeyklay
- **Head SHA**: 464e133f729b58232008740953368e57549ee1ca
- **Link**: https://github.com/sst/opencode/pull/24538
- **Size**: +1/-0 — single new row in `packages/web/src/content/docs/ecosystem.mdx:73`.

## Scope

Adds Sortie (https://github.com/sortie-ai/sortie) to the ecosystem table. Sortie is a ticket-driven agent orchestrator (Jira / GH Issues → parallel agent sessions) that recently shipped an opencode adapter alongside its existing adapters, per the linked changelog (`docs.sortie-ai.com/changelog/#1.9.0`).

## Specific findings

- `ecosystem.mdx:73` — single row insertion in the existing pipe table, between `CodeNomad` and the `---` separator. Column widths preserved with the same trailing-space padding the rest of the table uses; no alignment drift versus surrounding rows.
- Description is a one-liner ("Turn Jira and GitHub tickets into parallel OpenCode sessions") that matches the tone and length of neighbours like `CodeNomad`, `ocx`, `OpenWork`. Doesn't oversell, doesn't editorialize.
- Link target `github.com/sortie-ai/sortie` resolves and is a real repo (not a 404 placeholder). The PR body also links to a hosted changelog page that documents the opencode adapter — adequate provenance.
- No Astro frontmatter, schema, or table-cell escaping issues — pure additive change.

## Risk

Negligible. Pure docs, single line, no code/build/runtime impact, no rendering surprises (table format unchanged).

## Verdict

**merge-as-is** — third-party ecosystem entries are exactly the kind of low-friction contribution this table is designed to accept, and the PR follows the exact format already established by surrounding rows. No nits worth blocking on.
