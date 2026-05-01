# sst/opencode#25109 — docs: add BRHP to ecosystem plugins

- **PR**: https://github.com/sst/opencode/pull/25109
- **Head SHA**: `d50ce8ea6d94c8a0a81f98d8567af043159a1c8d`
- **Files**: `packages/web/src/content/docs/ecosystem.mdx` (+1/-0)
- **Verdict**: **merge-as-is**

## Context

Single-line docs addition appending one row to the ecosystem plugins table at `packages/web/src/content/docs/ecosystem.mdx:47`, listing `BRHP` (a structured/persistent planning plugin with local session state, `/brhp` commands, and a TUI sidebar) with link to `https://github.com/ZanzyTHEbar/brhp`. Author confirmed in the PR body it is docs-only with no code changes; the diff is exactly the markdown table row insertion at line 47, between the `opencode-conductor` and `micode` rows.

## What's right

- Format-conformant: pipe alignment matches the surrounding rows at `:44-49`, description fits the existing one-line summary style ("Structured, persistent planning with local session state, `/brhp` commands, and a TUI sidebar").
- Insertion point is alphabetically-adjacent and visually grouped with the other planning/workflow plugins (`opencode-conductor`, `micode`, `octto`, `opencode-background-agents`).
- The page itself invites such PRs (existing link to `awesome-opencode` in the surrounding context implies community-curated entries are welcome).
- Zero risk surface: docs-only, single line, no executable change.

## Risks / nits

- None substantive. Could optionally cross-link to `awesome-opencode` if BRHP is also listed there, but that's a non-blocking polish.

## Verdict

**merge-as-is.** Single-line, format-conformant, docs-only ecosystem table addition. Exactly the shape these PRs should be.
