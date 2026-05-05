# charmbracelet/crush PR #2800 — feat(tools): create an allow list for MCP tools

- Repo: `charmbracelet/crush`
- PR: #2800
- Head SHA: `3394b9fb8fd5`
- Author: `BrunoKrugel`
- Scope: 4 files, +83 / -11
- Verdict: **merge-after-nits**

## What it does

Adds a per-MCP-server `enabled_tools` allow-list to complement the existing `disabled_tools` deny-list, so that for verbose MCP servers (the PR uses `mcp-atlassian` with 73 tools as motivation) you can enable just the tools you want without enumerating the other 70 in `disabled_tools`.

Concrete changes:

- `internal/config/config.go:174-180` (context) — `MCPConfig` gets a new `EnabledTools []string` field with a JSON schema description "Allow list of tools from this MCP server. When set, only these tools are enabled".
- `internal/agent/tools/mcp/tools.go:144-181` — replaces the old `filterDisabledTools` helper with a renamed `filterTools(mcpCfg, tools)` that:
  1. If `len(EnabledTools) > 0`: keep only tools whose `Name` appears in `EnabledTools`.
  2. Then, if `len(DisabledTools) > 0`: drop any tools whose `Name` appears in `DisabledTools`.
  3. Returns the filtered slice.
- `updateTools()` (line 145-152) now does `mcpCfg, ok := cfg.Config().MCP[name]` and only calls `filterTools` when the server has config; the no-config case skips filtering entirely.
- `internal/agent/tools/mcp/tools_test.go` adds 5 sub-tests covering: no-filter, deny-only, allow-only, allow+deny intersection (allow=`{a,b}` ∧ deny=`{b}` → `{a}`), and allow with non-existent tool (→ empty).

## What's good

- **Solves a real problem.** Big MCP servers (Atlassian, Github, Linear, Notion) ship dozens of tools; allow-lists are the right primitive for "I only want `jira_get_issue` and `jira_search` from this 73-tool surface".
- **Allow-then-deny ordering is correct and intuitive.** `enabled_tools = {a, b}` ∧ `disabled_tools = {b}` → `{a}` matches user intuition that explicit deny wins over explicit allow. The 5th sub-test (`enabled and disabled both apply`) locks this in.
- **Non-breaking.** Existing configs that only use `disabled_tools` keep working unchanged; `enabled_tools` is opt-in via the `len(...) > 0` gate. JSON schema gets the new field as optional.
- **Helper renamed thoughtfully.** Old name was `filterDisabledTools`; new name `filterTools` reflects that it does both directions. The signature change from `(cfg *config.ConfigStore, mcpName string, tools []*Tool)` → `(mcpCfg config.MCPConfig, tools []*Tool)` is also a small win — the helper no longer needs to look up its own config, callers do that once and pass it in.
- **Test for the empty-allow-list case.** `enabled_tools: ["non_existent"]` → empty result is the right behaviour (a typo in the allow-list shouldn't silently expose all tools), and the test locks it in.

## Nits / discussion points

1. **Empty allow-list semantics conflate "not set" with "set to []"**. The JSON tag is `json:"enabled_tools,omitempty"` and the gate is `len(mcpCfg.EnabledTools) > 0`. So `enabled_tools: []` is treated the same as the field being absent (i.e. all tools allowed, then deny applies). Some users will write `enabled_tools: []` expecting "allow nothing" as a kill-switch. Either:
   - document explicitly in the schema description that `[]` means "no allow-list configured" (not "deny everything"), or
   - distinguish absent vs. empty, e.g. `EnabledTools *[]string` with the gate `mcpCfg.EnabledTools != nil`. Pointer-to-slice is uglier but unambiguous.

2. **No validation/warning for unknown tool names.** A typo in `enabled_tools: ["jira_get_isue"]` (missing `s`) results in zero tools being available from that server with no diagnostic. A startup-time log line listing allow-list entries that didn't match any discovered tool — `WARN mcp.<name>: enabled_tools entry "jira_get_isue" did not match any tool` — would catch this fast.

3. **Order of operations not documented.** The PR body shows the allow-list use case but doesn't spell out the allow-then-deny precedence. The 5-test suite is the only place that documents it. Worth one line in the schema description ("disabled_tools is applied after enabled_tools, so an entry in both is denied").

4. **No e2e / integration test against a real MCP server.** Unit tests cover the helper's pure logic, which is what matters here, but a `TestUpdateToolsAppliesAllowList` that exercises `updateTools()` end-to-end with a fake `cfg` would catch a future refactor that bypasses `filterTools` entirely.

5. **`filterTools` is now exported-by-name from a package called `mcp`** (lowercase `f`, so technically still package-private — confirmed). Just noting that if any external package wanted to share this filter logic, they'd need to copy it. Probably fine.

## Verdict
**merge-after-nits** — clean, well-tested implementation of an obviously-useful feature. Resolve the empty-vs-absent semantics question (#1) and add a startup warning for unmatched allow-list entries (#2) and this is shippable as-is.
