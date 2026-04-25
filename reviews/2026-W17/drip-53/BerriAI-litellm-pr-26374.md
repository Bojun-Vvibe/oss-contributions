# BerriAI/litellm#26374 — fix(mcp): do not prefix tool names when listing via scoped /mcp/{server_name}
**SHA**: `796844ee` · **Author**: stuxf · **Size**: +401/-25 across 2 files

## What
Fixes issue #22670: when an MCP client connects to a path-scoped URL
like `/mcp/{server_name}` (or `/{server_name}/mcp`), tool / prompt /
resource / resource-template listings should NOT be prefixed with the
server name, because the URL itself has already disambiguated which
server is in play. Currently the proxy unconditionally prefixes all
names with the server name (necessary for the aggregated `/mcp`
endpoint where multiple servers share a namespace), causing scoped
clients to see names like `github_onprem-create_issue` instead of
`create_issue`.

## Specific cite
- `litellm/proxy/_experimental/mcp_server/mcp_context.py:23-34`
  introduces a new `ContextVar` `_mcp_request_scoped_to_single_server`,
  defaulting to `False`. The choice of `ContextVar` (vs threading a
  param through every call site) is the right one here — it's
  transport-agnostic across stdio/SSE/HTTP-streamable, exactly as the
  docstring claims.
- `server.py:2496-2503` is the set-side, inside `extract_mcp_auth_context`:
  `if len(mcp_servers_from_path) == 1: _mcp_request_scoped_to_single_server.set(True)`.
  The `== 1` guard is correct — a multi-server scoped path would
  reintroduce ambiguity and prefixing should stay on.
- `server.py:1323-1336, 1471-1478, 1531-1538, 1589-1596` are the four
  read-sides (`_fetch_and_filter_server_tools`,
  `_get_prompts_from_mcp_servers`, `_get_resources_from_mcp_servers`,
  `_get_resource_templates_from_mcp_servers`). Each replaces the
  hard-coded `add_prefix=True` with
  `add_prefix_for_server = not _mcp_request_scoped_to_single_server.get()`.
  Symmetric and consistent — good.
- One concern: the `ContextVar` is set but never explicitly reset.
  In ASGI/Starlette the context is per-request so it's fine in
  practice, but a `try/finally` reset (or a token-based set/reset)
  would make the invariant explicit and survive any future refactor
  that runs handlers outside a request context (e.g. background
  tools listing for prefetch/cache).

## Verdict: merge-after-nits
Correct fix, well-targeted at the actual ambiguity-vs-disambiguation
boundary, applied symmetrically at all four list sites. The one nit
worth addressing before merge is wrapping the `set(True)` in a
token/reset pattern to make the per-request lifetime explicit rather
than relying on ASGI's context isolation. A test asserting the
aggregated `/mcp` endpoint still prefixes (regression-guard for the
default case) would also be valuable.
