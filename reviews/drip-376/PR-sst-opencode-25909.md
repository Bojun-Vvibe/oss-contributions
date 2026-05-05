# sst/opencode #25909 — feat(websearch): add Perplexity backend (default), keep Exa as alt

- Author: jliounis
- Head SHA: `916eb3aabe3d8969a202a0490442ef7d8d52015a`

## Files changed (top)

- `packages/opencode/src/tool/mcp-perplexity.ts` (+76 / -0, new)
- `packages/opencode/src/tool/websearch.ts` (+37 / -15)
- `packages/web/src/content/docs/providers.mdx` (+91 / -0)
- `packages/opencode/src/tool/registry.ts` (+5 / -1)
- `packages/core/src/flag/flag.ts` (+6 / -0)

## Observations

- `packages/core/src/flag/flag.ts:73-78` — gating logic is `!falsy("OPENCODE_DISABLE_PERPLEXITY") && !truthy("OPENCODE_DISABLE_PERPLEXITY") && (...)`. The double-negative dance (both `!falsy` and `!truthy` against the disable env) is hard to read and arguably wrong: if `OPENCODE_DISABLE_PERPLEXITY=1` then `truthy(...)` is true, `!truthy(...)` is false, gate closes — fine. But `!falsy(...)` against the *disable* env is a redundant check (anything not parseable as a falsy-literal passes); consider replacing the `&&` chain with a single `!truthy("OPENCODE_DISABLE_PERPLEXITY")` for the kill-switch.
- `packages/opencode/src/tool/mcp-perplexity.ts:33-46` — `Effect.die(new Error("PERPLEXITY_API_KEY is not set"))` will surface as a defect rather than a recoverable failure; `Effect.fail` would be more consistent with how `mcp-exa.ts` handles missing-credentials cases. The `usePerplexity()` predicate in `websearch.ts` already gates on key presence so the die is defensive, but a clean `fail` is friendlier for tool-caller error reporting.
- `packages/opencode/src/tool/websearch.ts:18-29` — annotation update correctly marks `livecrawl` and `type` as Exa-only and ignored under Perplexity. Good. But the JSON schema snapshot at `packages/opencode/test/tool/__snapshots__/parameters.test.ts.snap:430` still surfaces these to the model as enums; consider gating their visibility based on backend choice in a follow-up so the model doesn't waste tokens reasoning about ignored params.
- `packages/opencode/src/tool/registry.ts:285-291` — gating `WebSearchTool` on `OPENCODE_ENABLE_EXA || OPENCODE_ENABLE_PERPLEXITY` is correct, but note this implicitly registers the tool the moment a user exports `PERPLEXITY_API_KEY` system-wide, which is a behavior shift from prior "explicit opt-in via `OPENCODE_ENABLE_EXA`" semantics. Worth calling out in the docs page (`providers.mdx`) — currently it says "default when key is set" but doesn't flag the implicit-registration side effect.
- Docs at `packages/web/src/content/docs/tools.mdx:259-265` are clear about precedence; nice that `OPENCODE_DISABLE_PERPLEXITY=1` is documented as the override.

## Verdict

`merge-after-nits`

Solid feature with new module, docs, and snapshot updates aligned. Two nits worth addressing pre-merge: the redundant `!falsy && !truthy` double-check in `flag.ts:73-78`, and the `Effect.die` → `Effect.fail` swap in `mcp-perplexity.ts:34`. Schema-level hiding of Exa-only params under Perplexity backend is a fair follow-up and not blocking.
