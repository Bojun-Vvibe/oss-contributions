# google-gemini/gemini-cli PR #25989 — fix(mcp): handle hyphenated server names consistently in tool dispatch

- URL: https://github.com/google-gemini/gemini-cli/pull/25989
- Head SHA: `fa44ca56e9df655ea27358c1371aae8968b7c842`
- Diff: +88 / -1 across 2 files
- Author: external contributor (MukundaKatta)
- Closes: #25952

## Context / Problem

MCP tools register under their literal hyphenated namespaced names
(e.g. `mcp_hyphen-server_test-tool`), but Gemini models frequently
emit the function-call name with hyphens collapsed to underscores
(`mcp_hyphen_server_test_tool`). The literal lookup in
`ToolRegistry.getTool` missed and the call surfaced "tool not
found" — even when the right tool was registered. The bug shape
is "model hallucinates separator normalization, registry refuses
to meet it halfway."

## Design

Single new private method `resolveMcpToolNameLoose` at
`packages/core/src/tools/tool-registry.ts:840-866` plus a free
helper `collapseMcpSeparators(name)` at `:870-877`:

```ts
function collapseMcpSeparators(name: string): string {
  if (!name.startsWith(MCP_TOOL_PREFIX)) return name
  return MCP_TOOL_PREFIX + name.slice(MCP_TOOL_PREFIX.length).replace(/-/g, '_')
}
```

Lookup loop iterates `this.allKnownTools`, filters to
`DiscoveredMCPTool` instances, compares their
`collapseMcpSeparators(registeredName)` against the target's
collapsed form, and **refuses ambiguous matches**:

```ts
if (match) {
  // Ambiguous: refuse to guess.
  return undefined
}
match = registeredTool
```

The fallback only fires after the literal and legacy-alias
lookups both miss, and only on names with the `mcp_` prefix
(`tool-registry.ts:803-813`). Non-MCP tools are unaffected —
asserted by the `should leave non-MCP tools alone` test.

## Tests at `tool-registry.test.ts:289-322`

Three pins covering the contract surface:

1. `should resolve a hallucinated snake_case name to the registered
   hyphenated MCP tool` — registers `mcp_hyphen-server_test-tool`,
   asserts both the literal lookup and the collapsed
   `mcp_hyphen_server_test_tool` resolve to the same tool. This is
   the bug-fix pin.
2. `should return undefined when two MCP tools collapse to the same
   name` — registers `mcp_a-b_c` and `mcp_a_b-c` (both collapse to
   `mcp_a_b_c`), asserts the lookup returns `undefined`. This is
   the safety pin — refuse to silently dispatch on ambiguity.
3. `should leave non-MCP tools alone` — registers `mock-tool`,
   asserts `mock_tool` lookup is `undefined`. This is the
   blast-radius pin — the loose match must not bleed into non-MCP
   tool lookup.

The three pins together give a behavioral contract: "snake_case
shorthand for unique MCP tool names works; ambiguity refuses to
guess; non-MCP tools unchanged." That's the right shape.

## Strengths

- **The ambiguity refusal is the load-bearing decision.** Silent
  dispatch on `mcp_a_b_c` when two registered tools collapse to
  it would route a model's call to whichever tool the iteration
  saw first — non-deterministic, and the kind of bug that
  surfaces six months later as "the wrong tool ran my command."
  Returning `undefined` forces the model to retry with a
  disambiguating literal name.
- **Debug log on resolved fallback** at `tool-registry.ts:809-811`
  (`Resolved hyphen-insensitive MCP tool name "X" to "Y"`) keeps
  the underlying model-behavior bug visible in logs even after
  the dispatch succeeds. Good.
- **Prefix gate prevents collateral damage** — non-MCP tool names
  are returned as-is from `collapseMcpSeparators` (early return
  at `:872`), so the loose-match path can never widen into the
  general tool-name namespace.

## Risks / nits

1. **One-way collapse only** (hyphens → underscores), not
   underscores → hyphens. If a model ever emits
   `mcp_hyphen-server_test_tool` (mixed case) or
   `mcp_hyphen_server_test-tool`, the collapse target won't match
   `mcp_hyphen-server_test-tool`. Probably fine — the failure mode
   the PR fixes is "model collapses everything to underscores,"
   not "model partially preserves hyphens." But worth a one-line
   comment on the helper noting the asymmetry.
2. **O(n) lookup** — the loop iterates every registered tool on
   every fallback. For workspaces with hundreds of MCP tools that's
   a measurable cost on every miss. A separate
   `mcpToolsByCollapsedName` index built incrementally on
   register/unregister would make the fallback O(1) for the
   non-ambiguous case. Acceptable as v1 since the loop only fires
   on a literal-lookup miss, which should be rare.
3. **Three-element ambiguity** — the test pins the two-element
   case but not "three tools collapse to the same name." The loop
   handles it correctly (first ambiguity short-circuits to
   `undefined`) but a one-line test pin would be cheap insurance.
4. **No telemetry** — the debug log is dev-only. Counting how
   often the loose fallback fires in production would tell the
   maintainers which models hallucinate the most aggressively
   and inform whether the fix should eventually become
   "normalize on registration too."

## Verdict

`merge-as-is` — surgical fallback that meets a known model
behavior halfway, with the right safety property (ambiguity
returns `undefined`, never guesses), the right blast-radius gate
(`mcp_` prefix only, `DiscoveredMCPTool` instances only), and
three test pins that together describe the full contract. The
async-index optimization and the asymmetric-collapse comment are
both follow-ups, not blocks.

## What I learned

The right shape for a "model-behavior compatibility shim" is:
(a) gate it tightly to the affected namespace, (b) refuse to
guess on ambiguity, (c) log every time it fires, (d) test the
both-found / ambiguous / non-affected-namespace contract
explicitly. Skip any of (a)-(d) and the shim becomes a long-tail
debugging nightmare.
