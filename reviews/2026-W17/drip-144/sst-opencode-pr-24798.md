---
pr_number: 24798
repo: sst/opencode
head_sha: de629794aa4ccd6c39db190264de2ec081f7d461
verdict: merge-after-nits
date: 2026-04-28
---

# sst/opencode#24798 — feat: add support for named agent colors

**What changed.** +51/−6 across 6 files. Three layers:

1. **Schema admission** at `packages/opencode/src/config/agent.ts:20`: `Color` union extended with a third arm `Schema.Literals(["red", "blue", "green", "yellow", "purple", "orange", "pink", "cyan"])`. Description annotation at `:43` updated to mention "or other valid CSS colors".
2. **Theme map admission** at two parallel sites — `packages/app/src/utils/agent.ts:6-13` and `packages/ui/src/components/message-part.tsx:284-291` — adding the same 8 named-color → `var(--syntax-*)` token mappings to the `defaults`/`agentTones` records.
3. **Resolver liberalization** at `packages/opencode/src/cli/cmd/tui/context/local.tsx:92-100`: replaces the `theme[color as keyof typeof theme] as RGBA` blind-cast with a guarded `if (color in theme)` lookup, then falls back to `Bun.color(color, "hex")` parser when the input isn't a known theme key. Generated SDK types regenerated at `packages/sdk/js/src/v2/gen/types.gen.ts:1255-1318`.

Closes #24797.

**Why it matters.** Claude-Code-format subagent configs ship named colors like `"color": "red"` that previously failed schema validation (the union accepted only hex + theme tokens). Adding the literal arm + the runtime fallback closes the parity gap. The `Bun.color()` fallback also widens beyond the 8 named CSS colors to anything Bun parses (`"rebeccapurple"`, `"hsl(...)"` etc.) without needing further schema work.

**Concerns.**
1. **`purple → --syntax-property` and `pink → --syntax-property` collide,** as do `orange → --syntax-warning` and `yellow → --syntax-warning`, and `blue → --syntax-info` / `cyan → --syntax-info`. Three pairs of names render visually identically. That's a deliberate compromise (no purpose-built `--icon-agent-purple-base` token exists yet), but it means the named-colors UX promise is **8 names map to 5 distinct colors** — worth a one-line note in `agents.mdx:631` so users don't file "pink looks like purple" issues.
2. **Schema arm widens but `Bun.color()` widens further.** A user writing `"color": "rebeccapurple"` will fail schema validation upfront (rejected by the literal union) but would have worked at runtime via the `Bun.color()` fallback. Either narrow the resolver to only the 8 admitted names, or widen the schema to `Schema.String.check(...)` with a CSS-color validator. Current diff has the schema and resolver disagreeing on what's accepted.
3. **The `theme[color as keyof typeof theme] as RGBA` → `theme[color as keyof typeof theme]` rename at `local.tsx:97`** drops the trailing `as RGBA` cast. If `theme[key]` returns something other than `RGBA` for any of the 8 new keys, this would be a type-system regression silently caught only at runtime. Worth confirming the `theme` table shape includes all 8 new entries with `RGBA` values.
4. **No test added** for any of the three layers (schema-rejects-`"magenta"`, schema-accepts-`"red"`, resolver-falls-through-to-`Bun.color`). A 4-cell matrix in `config/agent.test.ts` (or wherever schema tests live) would pin the behavior.
5. **Generated SDK types regen at `types.gen.ts`** is correctly included — without it, downstream typed SDK consumers would silently lose the new arm.
6. **Two parallel `defaults`/`agentTones` records** (one in `app/src/utils/agent.ts`, one in `ui/src/components/message-part.tsx`) is a known duplication; this PR doesn't introduce it, but the 8-color admission has to land in both. A follow-up to extract a shared constants module would prevent these from drifting.

Useful UX win, ship after the schema/resolver-disagreement clarification (concern 2) and a brief docs note about the 5-distinct-colors reality (concern 1).
