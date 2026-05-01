# BerriAI/litellm #27016 — fix(mcp): run pre_call_tool_check on OpenAPI/local-registry path (VERIA-45 / VERIA-7)

- **Repo:** BerriAI/litellm
- **PR:** https://github.com/BerriAI/litellm/pull/27016
- **HEAD SHA:** `8ee599a`
- **Author:** stuxf
- **Verdict:** `merge-after-nits`

## What the diff does

Closes an authorization-bypass where MCP tools registered via the OpenAPI-to-MCP generator (the "local registry" path) were dispatched to `_handle_local_mcp_tool` *without* ever going through `pre_call_tool_check`. Only the managed-MCP-server path called the hook, so allowed/banned-tool checks, key/team tool permissions, parameter validation, and pre-call guardrails all silently no-op'd for OpenAPI-backed tools — the entire authorization surface was a coincidence of which dispatcher branch a tool fell into.

Fix at `litellm/proxy/_experimental/mcp_server/server.py:2104-2147` adds, after `local_tool = global_mcp_tool_registry.get_tool(name)` succeeds:

1. **A hard refusal when `mcp_server is None`** (raises HTTP 503 with the message naming the orphan/race condition). This is the load-bearing branch — without it the prior "skip checks if no server" implicit behavior would persist as a soft bypass. The status code (503) and the "Retry once the server is registered" guidance correctly express "this is a transient resolver-state problem, not a forbidden request."
2. **Sourcing `proxy_logging_obj` from `litellm.proxy.proxy_server`** (matching what `_handle_managed_mcp_tool` does). The kwargs-supplied path is `None` on the MCP entry point, so the fix imports it locally — without this the security checks would pass and then crash in `_create_mcp_request_object_from_kwargs` with `AttributeError`.
3. **Honoring guardrail-modified `arguments`** when `pre_call_tool_check` returns `{"arguments": ...}`. This matches the managed-path semantics so a guardrail that rewrites/redacts arguments is uniformly effective.

Tests at `tests/test_litellm/proxy/_experimental/mcp_server/test_openapi_tool_auth.py:1-220` cover the three behaviors that matter:

- `test_openapi_local_tool_runs_pre_call_tool_check` — the hook fires and `_handle_local_mcp_tool` is called once after.
- `test_openapi_local_tool_blocked_when_pre_call_check_raises` (HTTP 403) — the local handler must NOT be invoked.
- `test_openapi_local_tool_denied_when_server_not_resolvable` — HTTP 503, neither hook nor handler invoked.

## Why it's right

- **Diagnosis matches the security property.** "Authorization belongs at the dispatch boundary, regardless of which dispatcher you're routed to." The fix puts the gate *inside* `if local_tool:` so it executes on every dispatch path.
- **Fail-closed on `mcp_server is None`.** The natural temptation would be "skip the check if we don't have a server context." That's the prior bug. Refusing the call is the only correct disposition because `pre_call_tool_check` *is* the access-control gate; without a server you cannot evaluate it, so you cannot dispatch.
- **The blocked-when-raises test is the dispositive negative property** — it asserts the local handler is not awaited even after the hook raises. Without it a future "log and continue" change would silently re-open the bypass.
- **Cross-path symmetry**: arguments-rewrite-on-pre-call now works the same for managed and local tools. Guardrails authored against managed MCP tools port over without surprise.

## Nits

1. **`server_name=server_name or mcp_server.name`** at `:2138` — confirm `server_name` is in scope at this branch; if it's a parameter, document the expected non-None case explicitly so a future renamer doesn't drop the fallback.
2. **`from litellm.proxy.proxy_server import proxy_logging_obj`** is a function-level import on every call — module-level import (or a cached accessor) is cheaper and matches how other call sites in this file source it.
3. **The 503 message** mentions "the server is not available" but the actual condition is "the tool→server mapping has not finished initializing or the registry entry is orphaned." Two distinct conditions; including the tool name (which it does) plus "server registry uninitialized for this tool" would make operator triage faster.
4. **No timing-ordering assertion in the test** beyond "both were called" — the docstring acknowledges this, but a `Mock(side_effect=...)` capturing call order to a shared list would lock the property.
5. **Audit trail**: a `verbose_logger.warning` on the "mcp_server is None for local-registry tool {name}" branch would let operators detect the orphan-registry case in production without needing a 503-rate alert.
6. **The fix doesn't gate on `is_tool_allowed` at this layer** — confirm that check still runs upstream of `execute_mcp_tool`; if it doesn't and was relying on `pre_call_tool_check`, this fix may be the *first* time it executes for OpenAPI tools and any latent allowlist gaps will surface as 403s.

`merge-after-nits` — right primitive (gate inside `if local_tool:`, fail-closed on missing server, honor argument rewrites), three dispositive tests, but the comment-grammar and operator-observability nits should land in the same commit.
