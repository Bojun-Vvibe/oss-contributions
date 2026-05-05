# BerriAI/litellm#27196 — fix(mcp): include registration_endpoint in root /mcp OAuth discovery metadata

- **URL:** https://github.com/BerriAI/litellm/pull/27196
- **Head SHA:** `c8f6a6c4fe67a442efb7d293a30eeeea8bc1d2c5`
- **Reported by:** customer via Slack (no matching open GH issue)
- **Files touched:** 2 (+76 / −?)
  - `litellm/proxy/_experimental/mcp_server/byok_oauth_endpoints.py` (-39 + module-doc rewrite)
  - `tests/test_litellm/proxy/_experimental/mcp_server/test_byok_oauth_endpoints.py` (-24)
  - `tests/test_litellm/proxy/_experimental/mcp_server/test_discoverable_endpoints.py` (+71 regression test)

## Summary

`byok_oauth_endpoints.py` and `discoverable_endpoints.py` both registered handlers at `/.well-known/oauth-authorization-server` and `/.well-known/oauth-protected-resource`. The `_lazy_features.py` mount order put BYOK first, so its trimmed metadata payload (no `registration_endpoint`, generic `resource = base_url`) shadowed the correct discoverable payload at the root. MCP clients doing dynamic client registration (Claude Code among others) bailed at the discovery step with `"Incompatible auth server: does not support dynamic client registration"`. Per-server `/.well-known/oauth-authorization-server/{server}` paths were unaffected, which is why customers' per-server URLs kept working.

Fix: delete the duplicate handlers from `byok_oauth_endpoints.py`, leaving `discoverable_endpoints.py` as the single owner of the root well-known paths. Module docstring is rewritten to make the ownership boundary explicit.

## Line-level observations

- `byok_oauth_endpoints.py:9-15` — module-doc rewritten to explicitly assign `/.well-known/oauth-authorization-server` and `/.well-known/oauth-protected-resource` ownership to `discoverable_endpoints.py` and to call out that the discoverable variant always emits `registration_endpoint`. Documenting the ownership boundary directly in the module that *used to* own the path is the correct anti-regression hint for future changes.
- `:30-32` — drops the `from ...discoverable_endpoints import get_request_base_url` (no longer needed). Clean.
- `:592-630` (deletion) — removes both `oauth_authorization_server_metadata` and `oauth_protected_resource_metadata` handlers entirely. The deleted authorize-server payload was missing `registration_endpoint`, `scopes_supported`, `token_endpoint_auth_methods_supported`, and `refresh_token` grant — exactly the fields MCP DCR clients require. The deleted protected-resource payload returned `resource = base_url` instead of `base_url/mcp`, so even when discovery succeeded the resource indicator was wrong.
- `tests/.../test_byok_oauth_endpoints.py:-24` — drops the two now-stale tests (`test_oauth_authorization_server_metadata`, `test_oauth_protected_resource_metadata`). Removing the asserts that pinned the buggy shape is correct; reviewers should make sure no other test references these endpoints from the BYOK module. (`grep` not run here — flag for the maintainer.)
- `tests/.../test_discoverable_endpoints.py:1773-1843` — new regression test `test_root_well_known_handlers_not_shadowed_by_byok_router` is the most important file in the diff. It deliberately reproduces the production mount order (BYOK first via `app.include_router(byok_router)` at `:1814`, then discoverable at `:1815`) and asserts that the response (a) contains `registration_endpoint`, (b) does NOT contain BYOK's `/v1/mcp/oauth/authorize` token URL, and (c) returns `resource = base_url/mcp` for the protected-resource path. If a future PR re-introduces a shadowing handler in `byok_oauth_endpoints.py`, this test will fail with a clear, customer-grounded error message. Excellent regression coverage.
- The new test bracket-clears `global_mcp_server_manager.registry` in both `try`-prelude and `finally`, so it's safe to run in any order with the rest of the suite.

## Concerns

- The PR description's pre-submission checklist is unchecked (CI status, Greptile review). The author should fill those in before requesting maintainer review per the repo's contribution guide.
- The PR notes "no matching open GitHub issue" — worth filing one anyway so the customer-facing fix has a public reference.
- One unverified assumption: the regression test mounts a fresh `FastAPI()` and includes both routers manually. It does *not* run the actual `_lazy_features.py` registration code, so a future refactor to `_lazy_features.py` (e.g. a different mount order, or a third router added) would not be caught here. Worth a follow-up adding an integration test that exercises the real mount path.

## Verdict

**merge-after-nits**

## Rationale

Surgical deletion of a buggy duplicate, paired with a regression test that reproduces the production mount order and asserts the customer-visible symptoms. The fix is the smallest possible change for the bug, and the rewritten module-doc reduces the risk that someone re-adds the duplicate. Pre-submission checklist completion is the only real blocker; the integration-vs-unit-test gap and the missing GH issue are nice-to-haves.
