# Review: charmbracelet/crush#2782

- **PR:** charmbracelet/crush#2782
- **Head SHA:** `f243782466c732e761f6d791fbcf3c223cc8cb80`
- **Title:** fix(config): restore full shell expansion in MCP config values
- **Author:** meowgorithm

## Files touched (high-level)

- `README.md` — documents the new full shell expansion semantics (`$VAR`, `${VAR:-default}`, `$(command)`, quoting, nesting) across `command`, `args`, `env`, `headers`, and `url` for MCP config.
- `internal/agent/tools/mcp/init.go` — rewrites `createTransport` so each MCP transport branch (stdio, http, sse) calls the new resolver-aware accessors `m.ResolvedArgs(resolver)`, `m.ResolvedEnv(resolver)`, `m.ResolvedURL(resolver)`, `m.ResolvedHeaders(resolver)` and surfaces resolution errors instead of silently using unexpanded values.
- `internal/agent/tools/mcp/init_test.go` — adds `TestCreateTransport_URLResolution` exercising HTTP/SSE × success/failure × `$VAR` and `$(cmd)` expansion, plus a `shellResolverWithPath` helper that injects `PATH` so `$(echo ...)` / `$(cat ...)` work in test processes with otherwise-empty env.

## Specific observations

- `init.go:447-505`: the three switch arms (`config.MCPStdio`, `config.MCPHttp`, `config.MCPSSE`) are now structurally parallel — each resolves all dynamic fields up front, errors out cleanly, then constructs the transport. Good shape, no shared state surprises.
- `init.go` stdio branch: `m.ResolvedArgs(resolver)` and `m.ResolvedEnv(resolver)` are now called with an explicit resolver argument, replacing the old `m.ResolvedEnv()` zero-arg form. Worth confirming all call sites of the old `ResolvedEnv()` were migrated — this is the kind of rename where one forgotten caller silently keeps using a no-op resolver.
- `init.go` http/sse branches: previously `m.URL` was used directly with no expansion. The new `m.ResolvedURL(resolver)` enables `url: "https://$MCP_HOST/api"` and `url: "https://$(echo mcp.example.com)/events"`. Note the empty-URL check (`strings.TrimSpace(url) == ""`) now runs against the **resolved** URL — so an unset `$MCP_HOST` that resolves to `https:///api` will not trip the empty-URL check, but it _will_ trip an upstream resolver error (because unset vars are an error per the README). Good defense-in-depth ordering.
- `init.go` http/sse: `headers, err := m.ResolvedHeaders(resolver)` is now error-returning (vs. the old swallow-and-return-map signature). This is a behavioral improvement — previously a malformed header value would silently produce a broken header.
- `init_test.go`: the four-case matrix on URL resolution (http+success, sse+success, http+missing-var, sse+failing-cmd) is exactly the right pin for the seam. The error assertions check both `"url:"` prefix and the original token (`"$MCP_MISSING_HOST"`, `"$(false)"`) — that ensures error messages stay actionable and don't drift to a generic "expansion failed".
- `init_test.go` `shellResolverWithPath`: smart helper, gives future tests a path-injecting resolver factory. Could also be hoisted to a shared `internal/config/testutil` if more files start needing it.
- README hunk has a small typo: `"file-based secrets like work out of the box, so you can use values like \"$TOKEN\"\` and \`\"$(cat /path/to/secret/token)\"\``. The phrase "secrets like work out of the box" is missing a noun ("file-based secrets like X work out of the box…"), and the inline-code backticks are unbalanced (`"$TOKEN"\`` opens but doesn't close cleanly). This is a docs-only nit but renders weirdly on the project page.

## Verdict

**merge-after-nits**

## Reasoning

The core change is exactly the right shape: lift expansion from ad-hoc string substitution into a single `VariableResolver` seam invoked uniformly across `command`, `args`, `env`, `headers`, and `url`, with errors surfaced instead of swallowed. The new test matrix pins the URL seam at the transport boundary, which is where regressions would otherwise leak. The only blocker-grade nit is the README hunk's mid-sentence typo and unbalanced backticks (literally "secrets like work out of the box" — appears to be missing a noun and a closing backtick); fix those and merge. Worth a quick grep to confirm no leftover callers of the old zero-arg `ResolvedEnv()` / `ResolvedHeaders()` remain anywhere outside the touched file.
