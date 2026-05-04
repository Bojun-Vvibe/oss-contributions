# Review: google-gemini/gemini-cli #26247

- **Title:** fix: expand template vars in MCP stdio config
- **Head SHA:** `00a8540307657acec5d56f331709d039e3743ba4`
- **Size:** +67 / -4
- **Files touched:**
  - `packages/core/src/tools/mcp-client.test.ts`
  - `packages/core/src/tools/mcp-client.ts`

## Critique

Tight, well-scoped fix. Introduces `expandMcpServerConfigValue` that first substitutes `{{NAME}}` template variables from the merged env, then runs the existing `expandEnvVars` (`$NAME` / `$HOME` style). Applied to `command`, each `arg`, `cwd`, and each `env` value before constructing the `StdioClientTransport`.

Specific lines:

- `mcp-client.ts:1168-1178` — new `expandMcpServerConfigValue` helper. Regex `/\{\{(\w+)\}\}/g` only matches `\w` (alphanumeric + underscore), so dotted or hyphenated names won't expand. That's almost certainly intended (matches typical env-var naming). Falls back to leaving `{{NAME}}` literal if not in env — safer than throwing.
- `mcp-client.ts:2283-2294` — applies the expansion to `command`, `args`, `cwd` in addition to `env`. Previously only `env` was expanded, which meant `command: "{{HOME}}/.bun/bin/bun"` silently failed and the user got an `ENOENT` for a literal path containing `{{HOME}}`. Real bug, real fix.
- `mcp-client.ts:2294` — `args || []`. Note that the upstream `mcpServerConfig.args?.map(...)` already returns `undefined` when args is unset, so `args || []` just guards that. Fine.
- `mcp-client.test.ts:1996-2031` — new test covers all four expansion sites (`command`, `args`, `env`, `cwd`) with a single config and confirms `StdioClientTransport` is called with fully-expanded values. Good. The test mixes `{{HOME}}` and `$HOME` in the same args array, which is exactly the case that proves the two-pass strategy works.
- One small gap: no test for the "unknown variable" path (`{{NOT_SET}}` should remain literal). Would catch regressions if someone "improves" the regex to throw on misses.
- `process.env = originalEnv` in finally block — correct cleanup.

Backward compatible: previously-broken configs now work; previously-working configs still work because `$HOME` semantics are unchanged.

No banned strings.

## Verdict

`merge-as-is`
