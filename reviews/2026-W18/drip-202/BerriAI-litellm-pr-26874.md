# BerriAI/litellm #26874 — fix(mcp): add timeout to _descovery_metadata to prevent startup hang

- **Author:** gojoy (Guo Jix)
- **SHA:** `5e1b51e`
- **State:** OPEN
- **Size:** +4 / -1 in `litellm/proxy/_experimental/mcp_server/mcp_server_manager.py:1506-1512`
- **Verdict:** `merge-after-nits`

## Summary

Three-line fix that makes `_descovery_metadata` (sic) match the timeout
discipline of its two siblings in the same file. Pre-fix:

```python
client = get_async_httpx_client(llm_provider=httpxSpecialProvider.MCP)
```

Post-fix (`:1509-1512`):

```python
client = get_async_httpx_client(
    llm_provider=httpxSpecialProvider.MCP,
    params={"timeout": MCP_METADATA_TIMEOUT}
)
```

Without `params`, the underlying `get_async_httpx_client` falls through to the
`COMPLETION_HTTP_FALLBACK` default (the PR body cites 600s) — under Python
3.13 + uvloop on macOS that surfaces as an indefinite proxy startup hang
("Loading MCP Servers from config-----" never returns). The two adjacent
discovery helpers `_fetch_oauth_metadata_from_resource` (~`:1607`) and
`_fetch_single_authorization_server_metadata` (~`:1703`) already pass
`MCP_METADATA_TIMEOUT` via the same `params={"timeout": ...}` shape, so this
is a parity / drift fix, not a new policy decision.

## Reasoning

The fix is correct and the diagnosis is precise — RFC 9728 OAuth protected-
resource-metadata discovery is exactly the path where a misconfigured server
URL or a captive-portal upstream produces a long hang, and a 600s timeout in
the proxy's startup hot path is the wrong default. Three tightenings before
merge:

1. **No regression test was added,** which the PR template explicitly flags
   as "a hard requirement." The minimum useful regression: an httpx mock
   that asserts the constructed client carries `MCP_METADATA_TIMEOUT` rather
   than the fallback. Without it, a future refactor of `get_async_httpx_client`
   can silently regress all three call sites at once.

2. **The function name typo (`_descovery_metadata`) is reproduced in the
   diff** rather than fixed. Renaming to `_discovery_metadata` is a one-line
   change that prevents the "did you mean" failure mode on every grep ever
   done against this file. Worth bundling.

3. **The `params={"timeout": MCP_METADATA_TIMEOUT}` keyword shape relies on
   `get_async_httpx_client` accepting a free-form dict and forwarding it to
   `httpx.AsyncClient(...)`** — if a future signature change makes `params`
   a request-querystring concept (a common httpx footgun: `params` at the
   `httpx.AsyncClient` constructor is ignored, `params` at request time is
   querystring), the timeout will silently revert. The two sibling call
   sites have the same risk; a single helper like
   `get_async_httpx_client_with_timeout(MCP_METADATA_TIMEOUT, ...)` would
   eliminate the keyword-name footgun for all three.

Underlying-issue cross-references the PR body lists are real and worth
linking explicitly: #20715 (asyncio.wait_for + anyio TaskGroup conflict),
#22928 (TaskGroup race in MCP discovery), #23221 (3LO → 2LO discovery
fallback bug). This fix is orthogonal to all three but reduces the surface
on which they manifest as user-visible hangs.
