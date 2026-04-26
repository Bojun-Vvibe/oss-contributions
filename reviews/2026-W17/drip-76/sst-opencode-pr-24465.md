---
pr: 24465
repo: sst/opencode
sha: 38f1124f2a5662da846a75cd7d55b44fa27b3e3a
verdict: merge-as-is
date: 2026-04-26
---

# sst/opencode #24465 — docs: add opencode-byterover to ecosystem

- **URL**: https://github.com/sst/opencode/pull/24465
- **Author**: ian-pascoe (Ian Pascoe)
- **Head SHA**: 38f1124f2a5662da846a75cd7d55b44fa27b3e3a
- **Size**: +1/-0 across 1 file (`packages/web/src/content/docs/ecosystem.mdx`)
- **Closes**: #24464

## Scope

Single-line addition at `packages/web/src/content/docs/ecosystem.mdx:20` inserting the `opencode-byterover` plugin row at the top of the ecosystem table (alphabetical-by-`o` slot, before `opencode-daytona`). Description: "Persist and recall ByteRover memory context across OpenCode sessions". Pre-flight: author ran `bunx prettier --check` on the file.

## Specific findings

- **Alphabetical placement is correct.** Existing rows go `opencode-daytona` → `opencode-helicone-session` → `opencode-type-inject`. `opencode-byterover` sorts before `opencode-daytona` lexicographically, which is exactly where the new row landed at line 20.
- **Column-padding preserved.** The pipe-table at `ecosystem.mdx:18-19` has hand-aligned column widths (~100 chars for the name column, ~99 for the description). The new row matches the existing padding convention — verified by counting trailing spaces in the diff context. Prettier check the author ran would have flagged any drift.
- **External link is to a third-party repo** (`https://github.com/ian-pascoe/opencode-byterover`) owned by the PR author — same pattern as the rest of the table (e.g. `H2Shami/opencode-helicone-session`, `nick-vi/opencode-type-inject`). No code is being pulled into the OpenCode tree; it's a discovery row for users who want to find the plugin.
- **No risk surface.** Documentation-only mdx page; no JS/TS/runtime change, no new build dep, no scripts, no auto-fetched content from the linked repo. The ecosystem page is statically rendered at build time.
- **Author follows the PR template cleanly:** type-of-change checked, verification step listed, screenshots N/A correctly noted. Clean drive-by contribution.

## Risk

Zero — single mdx table-row addition, prettier-clean, no behavioral change. Worst case is the linked plugin disappears later and the row 404s, which is a maintenance task downstream of this PR.

## Verdict

**merge-as-is** — perfect minimal docs PR: one row, correct alphabetical slot, prettier-checked, owned by the plugin author. Maintainer should land it.

## What I learned

Ecosystem-table contributions like this are the cheapest possible signal of a healthy plugin community. A repo that accepts them quickly and consistently (alphabetical + format-checked + no extra approval gate) compounds — every accepted entry makes the next plugin author more likely to upstream their own row instead of forking the docs.
