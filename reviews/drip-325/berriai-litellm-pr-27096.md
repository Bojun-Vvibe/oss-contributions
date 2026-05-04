# BerriAI/litellm PR #27096 — fix(ui): coerce MCP cost values defensively before .toFixed (closes #27095)

- **PR:** https://github.com/BerriAI/litellm/pull/27096
- **Author:** Booyaka101
- **Head SHA:** `f880faf0` (full: `f880faf045c7841c2bb7cef7623e9944687e777f`)
- **State:** OPEN
- **Files touched:**
  - `ui/litellm-dashboard/src/components/mcp_tools/mcp_server_cost_config.tsx` (+25 / -21)
  - `ui/litellm-dashboard/src/components/mcp_tools/mcp_server_cost_display.tsx` (+28 / -34)
  - `ui/litellm-dashboard/src/utils/numberUtils.ts` (+33 / -0, new)
  - `ui/litellm-dashboard/src/utils/numberUtils.test.ts` (+59 / -0, new)

## Verdict

**merge-after-nits**

## Specific refs

- `ui/litellm-dashboard/src/components/mcp_tools/mcp_server_cost_display.tsx:11-19` — replaces the old `!== undefined && !== null` truthy check with `toFiniteNumber(...)` and rebuilds `toolCosts` as `Array<[string, number]>` filtered to finite-only values. This is the right shape: the render path no longer needs to defensively re-check or call `.toFixed` on potentially-string input.
- `ui/litellm-dashboard/src/components/mcp_tools/mcp_server_cost_config.tsx:131-162` (the IIFE block) — same coercion at every `.toFixed(...)` site in the config view. Wrapping the summary in `(() => { ... })()` works but is slightly heavier than extracting a `<CostSummary>` component or pre-computing the values above the JSX; readability nit, not blocking.

## Rationale

This is a clean defense-in-depth fix for a reproducible production crash (#27095) where JSONB-roundtripped numbers come back as strings and detonate `.toFixed`. Centralizing the coercion in `numberUtils.toFiniteNumber` plus 59 lines of unit tests is the correct fix shape — better than scattering `Number(x)` calls. Two small follow-ups worth raising in review: (1) the inline IIFE in `mcp_server_cost_config.tsx:131` is harder to scan than a hoisted `const summary = useMemo(...)` or a small subcomponent; (2) consider whether other `.toFixed(...)` call sites in the dashboard (spend tracking, key budgets) have the same string-from-JSONB exposure — if yes, a follow-up codemod is cheaper than waiting for the next #27095. Neither blocks the fix.

