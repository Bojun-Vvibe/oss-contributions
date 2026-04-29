# BerriAI/litellm#26748 — fix(mcp): sampling and elicitation flows

- URL: https://github.com/BerriAI/litellm/pull/26748
- Head SHA: `97553a2b602485995e7cf7f242e8048e0d54672f`
- Size: +1751 / −287, 12 files
- Verdict: **request-changes**

## Summary

Adds MCP sampling/elicitation/logging callback plumbing through `MCPClient.__init__` so upstream MCP servers can request LLM inference (sampling), user input (elicitation), or send log messages. New handlers at `litellm/proxy/_experimental/mcp_server/{sampling_handler,elicitation_handler}.py`, factory wiring at `mcp_server_manager.py:159-215` (`_create_sampling_callback` / `_create_elicitation_callback`), and `_create_mcp_client` passes them through for both stdio and HTTP/SSE transports. Test surface includes a custom MCP server fixture and live integration tests.

## Specific issues flagged — one is a bug

### 1. **BUG**: `session_ctx` constructed twice; first `ClientSession` leaks

At `litellm/experimental_mcp_client/client.py` (per diff lines 161-170 around `_execute_session_operation`):

```python
read_stream, write_stream = transport[0], transport[1]
session_ctx = ClientSession(read_stream, write_stream)        # <-- (1) constructed here
session_kwargs: Dict[str, Any] = {}
if self._sampling_callback is not None:
    session_kwargs["sampling_callback"] = self._sampling_callback
if self._elicitation_callback is not None:
    session_kwargs["elicitation_callback"] = self._elicitation_callback
if self._logging_callback is not None:
    session_kwargs["logging_callback"] = self._logging_callback
session_ctx = ClientSession(read_stream, write_stream, **session_kwargs)   # <-- (2) overwritten
session = await session_ctx.__aenter__()
```

The first `ClientSession(read_stream, write_stream)` constructor is called and immediately discarded. Depending on `mcp.ClientSession`'s `__init__` (it can subscribe to streams, attach background tasks, allocate request-id counters), this can:

- Double-subscribe to `read_stream` so the second session loses messages to the orphaned first one;
- Leave a `__aexit__` unbalanced if the constructor allocated anything that needs explicit cleanup (the first instance is GC'd without `__aexit__`);
- Race on shared internal mutable state in the MCP SDK.

Even in the harmless case it's wasted allocation. **Delete the first line.** Build `session_kwargs` first, then construct `session_ctx` once. This is a one-line fix and must happen before merge.

### 2. **REGRESSION**: `NPM_CONFIG_CACHE` injection now skipped when `server.env` is None/empty

At `mcp_server_manager.py:1184-1208` (per diff lines around `_create_mcp_client` stdio branch):

```python
resolved_env = (
    stdio_env
    if stdio_env is not None
    else (dict(server.env) if server.env else None)
)
# Ensure npm-based STDIO MCP servers have a writable cache dir.
if resolved_env is not None and "NPM_CONFIG_CACHE" not in resolved_env:
    resolved_env["NPM_CONFIG_CACHE"] = MCP_NPM_CACHE_DIR
```

Pre-PR: `dict(server.env or {})` always yielded a dict, so the `NPM_CONFIG_CACHE` check always ran. Post-PR: when `server.env` is `None` (the common case for "I don't need any custom env vars"), `resolved_env` becomes `None`, and the `is not None` guard skips the cache injection. **Result: any stdio MCP server registered without explicit `env:` config (i.e., most of them) loses the writable npm cache dir and breaks in containers** — exactly the failure mode the original code was defending against. This is the bug the comment on the next line warns about ("In containers the default (~/.npm or /app/.npm) may not exist or be read-only"). Either:

- Keep `resolved_env = dict(server.env or {})` so the cache injection always runs, OR
- Initialize `resolved_env = {"NPM_CONFIG_CACHE": MCP_NPM_CACHE_DIR}` when `server.env` is None.

The change to `None`-vs-`{}` was presumably to let the MCP SDK distinguish "inherit parent env" from "empty env", but that semantic shift was bundled into a sampling/elicitation PR with no test or release-note callout.

### 3. Diff includes 100+ lines of pure whitespace deletion in `client.py`

Diff lines 19-100 are purely `-` blank lines being removed (between the `import httpx`, `from pydantic import AnyUrl`, etc.). Mixing whitespace-only refactor with a feature change makes review noisy and `git blame` worse. Either ship as a separate cosmetic-only commit prior to this one, or revert the whitespace deletions.

### 4. `_create_sampling_callback` / `_create_elicitation_callback` re-import inside the closure on every call

At `mcp_server_manager.py:165-178`:

```python
async def _sampling_callback(context, params):
    from litellm.proxy._experimental.mcp_server.sampling_handler import (
        handle_sampling_create_message,
    )
    import litellm
    from litellm.proxy._experimental.mcp_server.server import (
        get_active_auth_context,
    )
    return await handle_sampling_create_message(...)
```

Imports inside the async callback execute every time the upstream MCP server requests sampling. `import` is cached after first call, so cost is negligible *but not zero* (sys.modules dict lookup + GIL). More importantly, this hides circular-import risk — if `sampling_handler.py` ever imports `mcp_server_manager` at module load, the deferred import here masks the cycle until first sampling request, which then crashes in production. Hoist the imports to module level if the cycle allows; if it doesn't, comment why the deferral is required.

### 5. `_create_*_callback` returns a closure that captures nothing — could be module-level

Both `_create_sampling_callback` and `_create_elicitation_callback` build a closure with no captured state. They're effectively module-level async functions wrapped in a no-op factory. Replace with plain `async def _sampling_callback(context, params)` at module level and pass `_sampling_callback` directly to `MCPClient(sampling_callback=_sampling_callback, ...)`. Eliminates one level of indirection and makes the call graph greppable.

### 6. Test surface uses live integration (`test_live_mcp.py`) — no unit tests for the callback factories

The two unit-test files target the *handlers* (`test_mcp_sampling_handler.py`, `test_mcp_elicitation_handler.py`) which is correct. But there's no unit test for `_create_sampling_callback` itself that asserts the closure correctly forwards `context`/`params` and pulls `default_mcp_sampling_model` from `litellm`. A 5-line `mock` test would have caught the duplicate-`session_ctx` bug at item 1.

## Why request-changes

Item 1 is a real concurrency-class bug in production-bound code. Item 2 is a stdio-server-startup regression for the most common configuration. Both must be fixed before merge.
