# BerriAI/litellm PR #27167 — fix: handle /mcp without trailing slash by adding explicit route

- Repo: `BerriAI/litellm`
- PR: #27167
- Head SHA: `6195d29cd539`
- Author: `krrish-berri-2`
- Scope: 2 files, +49 / -0
- Verdict: **merge-after-nits**

## What it does

Adds an explicit `/mcp` (no trailing slash) FastAPI route on the parent proxy app so that bare-`/mcp` requests are forwarded to the MCP streamable-HTTP handler directly, instead of getting Starlette's automatic `Mount("/mcp", ...)` redirect (`307 /mcp → /mcp/`) which **drops the request body** for clients that don't follow redirects.

`litellm/proxy/proxy_server.py:14962-14982` (new):

```python
@app.api_route(
    "/mcp",
    methods=["GET", "POST", "PUT", "DELETE", "PATCH", "OPTIONS", "HEAD"],
)
async def bare_mcp_route(request: Request):
    """Forward bare /mcp requests to the MCP streamable-HTTP handler."""
    from litellm.proxy._experimental.mcp_server.server import (
        handle_streamable_http_mcp,
    )
    scope = dict(request.scope)
    scope["path"] = "/mcp"
    return await _stream_mcp_asgi_response(
        handle_streamable_http_mcp, scope, request.receive
    )
```

Plus a regression-style test at `tests/test_litellm/proxy/_experimental/mcp_server/test_mcp_bare_route.py` that introspects `app.routes` and asserts there is exactly one explicit `/mcp` `Route` registered with `GET` and `POST` methods.

## What's good

- **Diagnoses a real interoperability bug.** Starlette's `Mount("/mcp", ...)` issues a 307 to add the trailing slash, and per RFC 7231 §6.4.7 a 307 must preserve method **and body**, but in practice many lightweight MCP clients (curl-based, hand-rolled httpx, Go `net/http` without `redirect=true`) either don't follow 307 at all on POST or follow it without re-emitting the body. Either way, the MCP handshake fails. Fixing this on the server side is the right move.
- **Body-safe path.** The handler reuses `_stream_mcp_asgi_response(handle_streamable_http_mcp, scope, request.receive)` and passes the raw `request.receive` callable through, so the ASGI message stream — including the request body — is forwarded unchanged. No `await request.body()` (which would buffer) and no scope rebuild beyond the `path` override.
- **Idempotent test.** `assert len(mcp_routes) == 1` will catch both regressions (route accidentally removed) and double-registration (route accidentally registered twice in a future refactor).
- **Local import inside the handler** correctly avoids a top-of-file import cycle with `_experimental/mcp_server/server`.

## Nits

1. **`OPTIONS` and `HEAD` semantics.** The new route advertises `OPTIONS` and `HEAD` but `handle_streamable_http_mcp` is the streamable-HTTP transport handler — it's not obvious it has sane `OPTIONS`/`HEAD` behavior. CORS preflight may surface this. Worth either narrowing methods to `["GET", "POST"]` (what the test asserts anyway) or adding a CORS-preflight test for `OPTIONS /mcp`.
2. **`scope = dict(request.scope)` then `scope["path"] = "/mcp"`** is a no-op rewrite (the path is already `/mcp` for this route). Either drop the `dict()` copy + reassignment or document why a future refactor might rely on the explicit set.
3. **Test asserts route presence, not behavior.** The test is good as a presence check but doesn't verify the redirect-avoidance contract. A `TestClient` request to `POST /mcp` (no slash) with a body and asserting `response.status_code != 307` and `request.body` was observed by the handler would catch a future regression where someone "fixes" the warning by reverting to the Mount default.
4. **Method list duplication.** The same method list is implicit in the comment ("so both /mcp and /mcp/ work identically"). If the underlying `Mount("/mcp", ...)` ever narrows its method set, the bare route will silently advertise more methods than the slashed route accepts. Low risk, but a one-liner that derives the method list from a shared constant would be more durable.

## Verdict
**merge-after-nits** — correct, body-safe fix to a real client-interop bug. Test is useful but could go one level deeper into actual request behavior. Ship it.
