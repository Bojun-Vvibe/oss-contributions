# charmbracelet/crush PR #2782 — fix(config): restore full shell expansion in MCP config values

- Head SHA: `40684228138303a922ff71a8f39dfe85fad30572`
- URL: https://github.com/charmbracelet/crush/pull/2782
- Size: +1632 / -386, 12 files
- Verdict: **merge-after-nits**

## What changes

Replaces the home-rolled `${VAR}` / `$(cmd)` parser previously used
for MCP config values with the embedded shell interpreter that Crush
already uses for the `bash` tool and hooks. Every MCP value site
(`command`, `args`, `env`, `headers`, `url`) now goes through the
same `config.VariableResolver` (`ResolvedArgs`, `ResolvedEnv`,
`ResolvedURL`, `ResolvedHeaders` on `MCPConfig`). `createTransport` in
`internal/agent/tools/mcp/init.go` is rewritten to call those
resolvers and surface their errors before constructing the underlying
`mcp.CommandTransport` / `mcp.StreamableClientTransport` /
`mcp.SSEClientTransport`. Adds substantial new tests in
`internal/agent/tools/mcp/init_test.go` covering URL resolution
across HTTP/SSE × success/failure × `$VAR` / `$(cmd)` shapes. Closes
#2334.

## What looks good

- Behavioral consolidation is exactly right. Having two independent
  shell-style expanders in the same binary (one for `bash` tool/hooks,
  one for MCP config) was a footgun: `${VAR:-default}` working in one
  and silently breaking in the other is the kind of bug users file as
  "MCP is broken" without ever realizing it's an expander mismatch.
- The error-propagation pattern in `createTransport`
  (`internal/agent/tools/mcp/init.go` lines ~447-505) is clean: each
  resolver call returns `(value, err)`, error short-circuits before
  the transport object is constructed. No partially-initialized
  transports leaked downstream. Compare to the legacy code where
  `m.URL` was used raw — a missing var produced a transport pointed
  at `https://$MCP_HOST/api` literal.
- The `TestCreateTransport_URLResolution` matrix (init_test.go line
  ~62) is the right shape: HTTP × success, SSE × success, HTTP ×
  unset var, SSE × failing `$(cmd)`. The error assertions check both
  `"url:"` prefix and the offending token (e.g. `"$MCP_MISSING_HOST"`,
  `"$(false)"`) which catches both error wrapping regressions and
  redaction regressions.
- `shellResolverWithPath` test helper (line ~16) explicitly hoists
  `PATH` from the host env into the resolver so `$(cat)`, `$(echo)`,
  `$(false)` actually find their binaries in a fresh test process.
  That's a subtle gotcha that will bite future test writers; calling
  it out as a helper with a comment is the right choice.

## Nits

1. README change (line ~298 of `README.md`) has a typo / partial
   sentence: `"works in `command`, `args`, `env`, `headers`, and `url`,
   so file-based secrets like work out of the box, so you can use
   values like "$TOKEN"` and `"$(cat /path/to/secret/token)"``." —
   "secrets like work out of the box" is missing a noun, and the
   backtick balance is off (`"$TOKEN"` opens with `"` and closes with
   `` ` ``). This needs a copy-edit before merge; it's user-facing
   docs.
2. README also says "Unset variables are an error; use `${VAR:-fallback}`
   to opt in to a default." That's a *behavior change* from the
   legacy expander, which (per the PR body) tolerated some unset-var
   cases. It's the right new default, but the PR body should call
   out the breaking change explicitly so anyone whose existing MCP
   config silently depends on an unset var getting "" gets warned.
3. The `createTransport` switch arms for `MCPHttp` and `MCPSSE`
   (lines ~470-505) are now ~95% identical — both build a `headers,
   err := m.ResolvedHeaders(resolver)` block, both wrap a
   `headerRoundTripper`, both return `&mcp.{Streamable,SSE}ClientTransport`.
   One private helper `func resolveHTTPClient(m, resolver) (url
   string, client *http.Client, err error)` would deduplicate.
4. Test shellResolverWithPath uses `os.Getenv("PATH")` directly. On
   CI containers with stripped `PATH`, `$(echo ...)` could still
   fail. Consider `if os.Getenv("PATH") == "" { t.Skip(...) }` as a
   guard, or hardcode `/usr/bin:/bin` as a floor.

## Risk

Medium-low. The runtime change touches the auth/headers/URL path of
every MCP transport, so any regression here breaks every MCP
integration silently or noisily. The "unset variables are now an
error" behavior change *will* break some users — but the failure mode
is immediate and self-explanatory rather than a malformed connection,
which is the right trade. Test coverage on the URL path is solid;
the `command` / `args` / `env` / `headers` paths look symmetrically
plumbed but aren't all individually unit-tested in the visible diff.
