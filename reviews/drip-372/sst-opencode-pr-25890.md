# sst/opencode#25890 — docs(ecosystem): add opencode-rexd-target plugin listing

- **URL:** https://github.com/sst/opencode/pull/25890
- **Head SHA:** `f2d8c701e69bfc4bf01f4cd6f338dfb00cee2576`
- **Files touched:** 1 (+1 / −0)
  - `packages/web/src/content/docs/ecosystem.mdx`

## Summary

Re-submission of a closed PR (#15833) that could not be reopened after rebase. Adds a single row to the ecosystem plugins MDX table for `opencode-rexd-target`, a remote-execution plugin that routes tools to remote servers running REXD over SSH (with local fallback, PTY support, patch/edit routing).

## Line-level observations

- `packages/web/src/content/docs/ecosystem.mdx:51` — single row insertion preserves alphabetical ordering against the surrounding rows (`opencode-notify`, `opencode-workspace`, `opencode-worktree`, `opencode-sentry-monitor`). The new row sits between `opencode-notify` and `opencode-workspace`, which is correct lexicographically.
- The cell uses inline markdown link `[REXD](https://github.com/samiralibabic/rexd)` inside the description column. Other rows in the table use plain text descriptions; only the first column uses inline links. Mild stylistic deviation, but the table renders fine in MDX and the extra link adds genuine value (REXD is a separate repo the reader needs to find).
- Description length (~210 chars) is on the long end for this table — adjacent rows are ~80–100 chars — but stays on one line in the MDX source and the surrounding rows already use long descriptions (e.g. `opencode-background-agents` at ~95 chars).
- No image / no JS / no schema change; pure additive docs.

## Verdict

**merge-as-is**

## Rationale

One-line additive docs change, alphabetical ordering preserved, both linked repos resolve. The author has done the rebase work that previously blocked #15833 and the PR description explicitly calls out merge-cleanness against current `dev`. Stylistic nits (inline link in description column, slightly long cell) are not worth blocking on for a docs-only change.
