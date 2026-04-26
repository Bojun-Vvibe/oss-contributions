---
pr: 26534
repo: BerriAI/litellm
sha: d55ee565b7805f5f7944835c0b8d1276901fb1b6
verdict: request-changes
date: 2026-04-26
---

# BerriAI/litellm #26534 — [WIP] Lazy-load optional feature routers on first request

- **Author**: Michael-RZ-Berri
- **Head SHA**: d55ee565b7805f5f7944835c0b8d1276901fb1b6
- **Size**: ~904 diff lines, primarily new module `litellm/proxy/_lazy_features.py` (316 lines) plus deletions across `litellm/proxy/proxy_server.py`.
- **Status**: marked `[WIP]` by the author.

## Scope

Memory footprint of `litellm` proxy ballooned because every optional feature module (MCP, agents, vector_stores, scim, cloudzero, vantage, policies, guardrails, claude-code-marketplace, etc.) is unconditionally imported at startup, dragging in their Pydantic schemas, FastAPI Dependants, and TypedDict metaclass machinery. The PR introduces a `LazyFeatureMiddleware` ASGI middleware that imports + registers each feature router only when an HTTP request first matches one of the feature's `path_prefixes`. Author reports ~700MB savings on a two-worker docker deployment.

## Specific findings

- `litellm/proxy/_lazy_features.py:62-71` — `LazyFeature(name, module_path, path_prefixes, register_fn)` is a clean dataclass. Good factoring. `register_fn` defaults to `_include_router("router")` and an `_mount_app(prefix, "app")` factory exists for ASGI sub-app mounts (used by MCP). Fine.
- `litellm/proxy/_lazy_features.py:271-279` — middleware does a per-request linear scan over `self._features` for prefix match. With ~17 features and `O(prefixes)` startswith checks, this is `O(features × prefixes)` per request. After all features are loaded, the early `if feat.module_path in self._loaded: continue` short-circuits — but the for-loop itself still runs. Once loaded, this is sub-microsecond, so fine.
- `litellm/proxy/_lazy_features.py:281-313` — `_load` uses a per-feature `asyncio.Lock` from `self._locks.setdefault(...)`. The lock dict isn't itself protected — if two requests arrive simultaneously for *different* unloaded features, the `setdefault` calls are safe (single-threaded event loop), but for the *same* feature, both coroutines call `setdefault` and both get the same `Lock` instance. Good. The double-checked `if feat.module_path in self._loaded: return` inside the lock is correct.
- `litellm/proxy/_lazy_features.py:286-294` — `loop.run_in_executor(None, importlib.import_module, ...)` offloads the import to a thread to avoid blocking the event loop on a 1-3s MCP import. **Concern**: Python module imports hold the import lock. Concurrent imports across threads (executor thread + main thread doing other imports as a side effect of request processing) can deadlock or serialize. Worth confirming the default `ThreadPoolExecutor` doesn't starve under load while a slow MCP import is in flight.
- `litellm/proxy/_lazy_features.py:294` — `feat.register_fn(self._fastapi_app, module)` mutates `app.router.routes` from inside an async context. The PR comment says "register_fn must stay on the loop thread since it mutates app.router.routes" — good awareness. But there is **no test** demonstrating thread-safety of this mutation against concurrent in-flight requests for *other* features.
- `litellm/proxy/_lazy_features.py:303-313` — failure handling is "mark loaded anyway, log a warning, requests will 404 until restart." **This is the biggest concern.** A transient import failure (e.g., a missing optional dep that gets installed via runtime hook, a circular-import race during the first concurrent burst, an `ImportError` from a backend the operator hadn't configured yet) becomes a permanent 404 with no recovery short of pod restart. At minimum, transient `ImportError`s should retry on a backoff, or the failure should surface as a 503 with a `Retry-After` rather than a silent 404 forever.
- `litellm/proxy/_lazy_features.py:91-95` — the `policies` feature uses prefix `/policy/` (with trailing slash) "so this doesn't also match `/policies/...` paths, which belong to policy_engine below." This kind of overlapping prefix discrimination is fragile — the *order* of `LAZY_FEATURES` plus the trailing-slash convention is the only thing keeping these straight. A regression test that hits `/policy/foo`, `/policies/bar`, and `/policies/resolve` and asserts which feature module gets loaded would prevent silent miscategorization.
- `litellm/proxy/_lazy_features.py:296-298` — `self._fastapi_app.openapi_schema = None` invalidates the cached OpenAPI on every load. Correct, but a high-traffic proxy that lazy-loads a feature mid-flight will rebuild the OpenAPI schema on the next `/openapi.json` request. The module docstring mentions UI form-builder code (`check_openapi_schema.tsx`) introspecting the schema. Operators using such tools post-deploy need a documented `warm` endpoint or boot script.
- `litellm/proxy/proxy_server.py:235-399` — large block of `from litellm.proxy.* import router as ..._router` deletions. Verify (a) none of these modules have `import`-time side effects (registering a global config, starting a background task, populating a registry like `global_mcp_tool_registry` or `global_agent_registry`) that the proxy depended on at startup. The diff at line 341-342 explicitly removed `from ... import global_mcp_tool_registry` and `from ... agent_registry import global_agent_registry` — these registries were likely populated as a side effect of import. If anything in the always-on code path refers to these globals before a lazy-load happens, behavior changes silently.

## Risk

**High.** The architectural direction is right (lazy-load is the standard fix for fat startup), but: (a) WIP status, (b) silent-permanent-404 on transient import failure, (c) deleted import-time side-effects (`global_mcp_tool_registry`, `global_agent_registry`) without explicit handling, (d) zero tests in the diff slice I inspected, (e) prefix-overlap convention is fragile and unencoded.

## Verdict

**request-changes** — fix the silent-404-on-transient-failure path (retry, or 503-with-retry-after), audit and explicitly handle the import-time side-effect deletions (the registries), add tests for prefix-overlap routing and concurrent first-load, and lift WIP. The 700MB savings is real and worth the work, but this PR cannot ship as-is.
