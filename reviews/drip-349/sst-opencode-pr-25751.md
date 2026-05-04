# sst/opencode#25751 — docs: add hiai-opencode to ecosystem plugins table

- **URL**: https://github.com/sst/opencode/pull/25751
- **Head SHA**: `625c202149c2`
- **Diffstat**: +1 / -0
- **Verdict**: `merge-after-nits`

## Summary

Single-row addition to `packages/web/src/content/docs/ecosystem.mdx` listing the third-party `hiai-opencode` plugin alongside the existing ecosystem entries.

## Findings

- `packages/web/src/content/docs/ecosystem.mdx:20` — new row inserted at the top of the table (above `opencode-daytona`). The table is otherwise alphabetically ordered by name (`opencode-daytona`, `opencode-helicone-session`, `opencode-type-inject`, ...), so `hiai-opencode` belongs at the top by `h` < `o` — alphabetical placement is correct.
- Description text "Canonical 12-agent model with bundled skills, MCP, LSP, and ralph-loop" — the phrase "12-agent model" is opaque to a first-time reader; a short rephrase like "12-agent preset bundling skills, MCP, LSP, and ralph-loop" would scan better. Not blocking.
- Pipe alignment / column padding matches the surrounding rows (verified by the diff context).
- Link target `https://github.com/HiAi-gg/hiai-opencode` resolves to a real repo per the PR author; reviewer should spot-check it's not abandoned before merging, since ecosystem entries effectively endorse external code.

## Recommendation

Plumbing is fine. Tighten the description and confirm the upstream repo is maintained, then ship.
