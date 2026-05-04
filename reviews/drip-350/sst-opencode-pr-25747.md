# sst/opencode#25747 — Add opencode-permission-guard to ecosystem plugins

- **URL**: https://github.com/sst/opencode/pull/25747
- **Head SHA**: `f159b5142850`
- **Diffstat**: +1 / -81
- **Verdict**: `request-changes`

## Summary

Intends to add a single ecosystem row for the third-party `opencode-permission-guard` plugin (`StuartGa/opencode-permission-guard`), but the diff actually deletes 81 lines from `packages/web/src/content/docs/ecosystem.mdx` — wiping the entire Plugins table, the Projects table, and the page frontmatter — and replaces them with one bare row.

## Findings

- `packages/web/src/content/docs/ecosystem.mdx:1-81` — the diff hunk replaces lines 1-81 (the `---\ntitle: Ecosystem\n---` frontmatter, the intro paragraph, the `awesome-opencode` callout, the entire Plugins table with ~35 entries including `opencode-daytona`, `opencode-firecrawl`, `opencode-sentry-monitor`, etc., the Projects table with 11 entries including `kimaki`, `opencode.nvim`, `OpenChamber`, `OpenCode-Obsidian`) with a single line for `opencode-permission-guard`. This is almost certainly an unintentional rebase / merge-conflict resolution gone wrong, not a deliberate intent.
- The PR description says "Adds opencode-permission-guard to the ecosystem plugins list" — the author clearly intended +1 / -0, not +1 / -81. The actual change would silently remove every other ecosystem entry on merge.
- The plugin itself looks reasonable (EACCES/EPERM monitoring, native OS notifications, focus detection, 10s cooldown, JSON config, en/es i18n) and would be a fine ecosystem addition if the diff matched intent.

## Specific code observations

- Line 1 of the new file is `[opencode-permission-guard](https://github.com/StuartGa/opencode-permission-guard)` with no surrounding `## Plugins` header, no frontmatter, no table header row — the page would not render as a table even after merge.
- No frontmatter means the doc page would lose its `title: Ecosystem` and break the docs nav.

## Recommendation

`request-changes`. Author needs to rebase onto current `main`, drop their merge resolution, and submit a clean +1 / -0 row appended to the existing Plugins table (matching the style of #25756 / `opencode-firecrawl`). Cannot merge as-is — would silently delete the entire ecosystem page content.
