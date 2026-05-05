# charmbracelet/crush #2800 — feat(tools): create an allow list for MCP tools

- **Head SHA:** `3394b9fb8fd5317058b8abcea967cd7aa28f0ddc`
- **Author:** @BrunoKrugel
- **Verdict:** `merge-after-nits`
- **Files:** +83 / −11 across `internal/agent/tools/mcp/tools.go`, `internal/agent/tools/mcp/tools_test.go`, `internal/config/config.go`, `schema.json`

## Rationale

Adds a per-MCP-server `enabled_tools` allow list complementing the existing `disabled_tools` deny list. The implementation in `internal/agent/tools/mcp/tools.go:7-37` (the rewritten `filterTools`) applies allow-list first, deny-list second, which is the right composition order: an explicit `enabled_tools` set narrows the universe, then `disabled_tools` can carve exceptions out of that. The new test `TestFilterTools` at `internal/agent/tools/mcp/tools_test.go:75-119` is well-structured (5 subtests using `t.Parallel()`) and explicitly exercises the precedence question — `t.Run("enabled and disabled both apply", ...)` at lines 105-113 covers a server config with `EnabledTools: [a,b]` and `DisabledTools: [b]` and asserts only `a` survives, which is the correct semantics.

The config-struct change in `internal/config/config.go:177` adds the `EnabledTools []string` field with a clean `jsonschema:"description=..."` tag, and the corresponding `schema.json:267-276` regeneration is consistent. The migration from the old `filterDisabledTools(cfg, name, tools)` to the new `filterTools(mcpCfg, tools)` correctly hoists the `cfg.Config().MCP[name]` lookup out of the filter and into the caller (`updateTools` at lines 7-13), which is a small but real readability win — the filter is now a pure function of `(MCPConfig, []*Tool)` and is independently testable, which is exactly why the new test could be written cleanly.

**Nits, in order of importance:**

1. **`schema.json:273` description is truncated:** `"Allow list of tools from this MCP server. When set"` — the sentence cuts off mid-clause. The `jsonschema` tag in `config.go:177` reads `"description=Allow list of tools from this MCP server. When set, only these tools are enabled,example=get-library-doc"`, and the `,` after "set" is being interpreted as the tag-separator rather than as English prose. Either escape the comma in the struct tag or split the description into a single comma-free sentence so the schema regen produces complete prose. (Same root cause exists in the existing `disabled_tools` description but at least *that* sentence ends at "disable" — the new one is mid-sentence, which will look broken in any tool that surfaces JSON Schema descriptions to users.)
2. **No test for the empty-allow-list edge case.** `TestFilterTools` covers `EnabledTools: []` implicitly via the "no filters returns all tools" subtest (since `len(mcpCfg.EnabledTools) > 0` is false), but the behavioral question worth pinning is: if a user *explicitly* sets `enabled_tools: []` in their config (intending "no tools"), they currently get *all* tools. That's surprising and probably wrong. Two options: (a) treat empty-but-present `enabled_tools` as "deny everything", or (b) document that `enabled_tools: []` is equivalent to omitting the key. Either is fine; the current behavior (silently equivalent to omission) should at least have a test pinning it as intentional.
3. **Naming consistency.** The deny list is `disabled_tools` (past participle) but the allow list is `enabled_tools` (also past participle) — fine, symmetric. But the more conventional names in this space are `allow`/`deny` or `include`/`exclude`. Not worth changing now (consistency with `disabled_tools` outweighs naming purity), but worth a one-line comment in the struct that "enabled" semantically means "allow-list, not opt-in default".
4. **No documentation update.** I'd expect a README/docs change describing how the two lists compose. The struct tag's prose isn't enough — a user reading the config docs needs to know "if I set both, allow runs first, deny runs second". The test pins the behavior; the docs should describe it.

Otherwise this is a clean small feature with proper test coverage. Land after the `schema.json` truncation fix (#1) at minimum.
