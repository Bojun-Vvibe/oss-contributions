# sst/opencode #25862 — docs(ecosystem): add opencode-smart-session-picker

- Head SHA: `ad9d3e30b7e8a0690c104b5b39f9f9e02d9ad102`
- Diff: +1 / -0 across 1 file (`packages/web/src/content/docs/ecosystem.mdx`)

## Findings

1. Single-row addition at `packages/web/src/content/docs/ecosystem.mdx:55` inserts the `opencode-smart-session-picker` row in alphabetical-by-feature-area order between the Firecrawl row at `:54` and the section separator at `:56`. Markdown table column widths are preserved (88-char name column, 100-char description column) — `git diff --check` will be clean.
2. Description (`Better search via fzf and hybrid (vector + BM25) through opencode sessions`) accurately reflects the linked repo's README (fzf for fuzzy + llama-server-backed embedding for semantic) — not aspirational marketing.
3. The link target is a third-party repo (`Techie5879/opencode-smart-session-picker`) — same trust model as every other ecosystem-table entry in this file; no new policy precedent.
4. Author self-identifies as the plugin creator in the PR body. This is consistent with the pattern of other ecosystem rows (Firecrawl, Sentry-Monitor, worktree all added by their maintainers).

## Verdict

**Verdict:** merge-as-is
