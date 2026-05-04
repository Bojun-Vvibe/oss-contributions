# Review: BerriAI/litellm #27112

- **Title:** feat(model_prices): add ai21 jamba-mini-2 and dated jamba-large-1.7 aliases (#27094)
- **Head SHA:** `7db78fc61ae67b9ef554cd5d5f21191aaee9095b`
- **Scope:** +60 / -0 across 2 files (model_prices_and_context_window.json + backup)
- **Drip:** drip-338

## What changed

Three new ai21 model entries are added to the price/context table:

1. `jamba-large-1.7-2025-07` — dated alias for the existing
   `jamba-large-1.7` family (input 2e-06 / output 8e-06, 256K context).
2. `jamba-mini-2` — new floating alias (input 2e-07 / output 4e-07, 256K
   context).
3. `jamba-mini-2-2026-01` — dated companion to `jamba-mini-2`.

All three use `litellm_provider: ai21`, `mode: chat`,
`supports_tool_choice: true`, with input/output/max all 256000.

## Specific observations

- `model_prices_and_context_window.json` lines ~22025-22075 — the dated
  `jamba-large-1.7-2025-07` entry inherits the same pricing as the
  existing `jamba-large-1.7` block above. Confirm with ai21's pricing
  page that the snapshot price didn't drift; dated aliases sometimes
  retain older pricing.
- `model_prices_and_context_window.json` ~22045-22075 — `jamba-mini-2` and
  `jamba-mini-2-2026-01` share identical numbers with `jamba-mini-1.6`
  (2e-07 / 4e-07). Suspicious — generation 2 of a model usually has
  different pricing. Worth a citation in the PR body to ai21's price
  page rather than a blind copy from the previous mini entry.
- `litellm/model_prices_and_context_window_backup.json` matches the root
  file diff line-for-line — good, the dual-write convention is honored.
- All three entries omit `supports_function_calling`,
  `supports_response_schema`, and `supports_system_messages` even though
  the existing `jamba-mini-1.6` block above also omits them. Consistent,
  not new drift, but the ai21 jamba family does support tools (already
  flagged via `supports_tool_choice`) — confirm whether the broader
  capability flags should also be set.
- Issue reference `(#27094)` is in the title; PR body presumably has the
  source link. Good.

## Risks

- Wrong pricing on a model gets quoted to users via cost reporting until
  the next release patches it. Verify the numbers against the upstream
  source before merging.
- No code changes, no test surface affected. Pure data file update.

## Verdict

**merge-after-nits** — confirm the three pricing rows against ai21's
published numbers (don't assume same as 1.6 mini / 1.7 large), then
merge.
