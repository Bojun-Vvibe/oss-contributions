# BerriAI/litellm PR #27244 — fix(proxy): preserve HTTP operations when injecting WebSocket stubs into OpenAPI schema

- URL: https://github.com/BerriAI/litellm/pull/27244
- Head SHA: `85d4d96c1bf1d823b1c66d5464768d568302b3f0`
- Size: +152 / -31

## Summary

Fixes a real production-class regression where `get_openapi_schema()` was unconditionally **overwriting** any existing path entry when injecting a synthetic GET stub for each `APIWebSocketRoute`. If a WebSocket route shared its path with an HTTP operation (e.g. a POST `/v1/responses` plus a websocket on `/v1/responses`), the entire HTTP operation vanished from the Swagger UI and from the served `/openapi.json`. Linked case `2026-05-05-madhu-swagger-responses-missing` in the test docstring confirms this hit responses-API users in v1.82.3.

## Specific findings

- `litellm/proxy/proxy_server.py:1060-1110` — extracted `_inject_websocket_stubs_into_openapi_schema(openapi_schema, websocket_routes)` helper. Two key changes inside:
  - `:1101` — `path_entry = openapi_schema["paths"].setdefault(base_path, {})` replaces the previous `openapi_schema["paths"][base_path] = { "get": {...} }` reassignment. `setdefault` returns the existing dict if present (preserving any `post`/`put`/`delete`/etc. keys) or a fresh `{}` if not. Correct fix for the bug.
  - `:1102-1110` — `if "get" not in path_entry: path_entry["get"] = { ... }` defends against the GET-vs-WebSocket collision case as well. Right call — a real GET should always win over the synthetic stub.
- `litellm/proxy/proxy_server.py:1116-1120` — caller refactored to:
  ```python
  openapi_schema = _inject_websocket_stubs_into_openapi_schema(
      openapi_schema, websocket_routes
  )
  ```
  Clean separation; the helper is now unit-testable in isolation (which the test suite exercises).
- `tests/test_litellm/proxy/test_openapi_schema_validation.py:144-249` — adds 4-case `TestWebSocketStubInjection`:
  - `:158-185` `test_websocket_stub_does_not_clobber_existing_post` — direct regression pin: shared path `/v1/responses` with both POST + WebSocket → both must remain. Asserts `operationId == "responses_api"` survives and `get.tags == ["WebSocket"]` is added.
  - `:188-204` `test_websocket_stub_added_when_path_is_new` — preserves prior behavior for WebSocket-only paths.
  - `:207-228` `test_websocket_stub_skipped_when_existing_get` — pins the GET-collision policy.
  - `:230-249` `test_responses_post_routes_registered_on_router` — sanity check that the three POST routes (`/v1/responses`, `/responses`, `/openai/v1/responses`) are still wired on the responses router. Guards against the source-of-truth side of the same bug.
- The `SimpleNamespace`-based `_make_fake_ws_route` at `:152-156` correctly stubs only `path`, `name`, `dependant=None` — matching the attributes the helper actually accesses.

## Concerns / nits

- The PR title truncates with `…` in the GitHub UI but the diff is self-contained.
- Extracted helper has no type hints on the return value beyond `dict` — minor; matches surrounding style.
- The `try/except (AttributeError, TypeError)` at `:1085-1093` retains the prior silent-swallow behavior for query-param extraction. Not changed, not in scope.
- One missed opportunity: the helper doesn't log a warning when it skips a stub due to existing GET — would help observability if a maintainer ever wonders "why isn't my WebSocket showing in Swagger". Not blocking.

## Verdict

`merge-as-is` — clean targeted fix for a real bug, with thorough regression coverage that pins the exact failure mode plus the GET-collision edge case. Ship it.
