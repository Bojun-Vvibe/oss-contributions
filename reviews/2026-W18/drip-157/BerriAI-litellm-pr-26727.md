# BerriAI/litellm PR #26727 — Revert "[Feat] Lazy-load optional feature routers on first request"

- **PR**: https://github.com/BerriAI/litellm/pull/26727
- **Author**: krrish-berri-2
- **Merged**: 2026-04-29 (post-#26534 revert)
- **Head SHA**: `aafae3e185f2`
- **Size**: +105/-609 across 5 files
- **Verdict**: `merge-as-is`

## Context

#26534 ("[Feat] Lazy-load optional feature routers on first request") landed
a `litellm/proxy/_lazy_features.py` registry plus a `LazyFeatureMiddleware`
in `litellm/proxy/proxy_server.py` that deferred ~25 optional feature router
imports (guardrails, agents, vector-stores, MCP server, evals, anthropic
passthrough, etc.) until the first request whose path matched a registered
prefix. The headline benefit was a claimed ~700 MB idle-memory saving for
deployments that don't use the optional features and a faster proxy startup;
the cost was a multi-second first-request latency on each lazy-loaded
prefix, OpenAPI-schema-omits-routes-until-warm semantics, and a
prefix/suffix matcher whose correctness was load-bearing on a hand-curated
deny list (`/policies` vs `/policy/`, `/v1/vector_stores` vs
`/vector_store/`, etc.). This PR is a clean revert.

## What changed (verified against the diff)

1. **Deletion of the registry entirely** (`litellm/proxy/_lazy_features.py`,
   `+0/-307`): the whole `LAZY_FEATURES` tuple plus `LazyFeature` dataclass,
   `_include_router`/`_mount_app` factories, and the
   `LazyFeatureMiddleware` ASGI shim are removed.

2. **Eager imports restored at the top of `proxy_server.py`** (lines
   235-431 of the diff): every gated module is back as a top-level
   `from litellm.proxy.X import router as Y` — visible in the diff at
   imports including `mcp_byok_oauth_router`, `mcp_discoverable_endpoints_router`,
   `mcp_rest_endpoints_router`, `mcp_app`, `global_mcp_tool_registry`,
   `a2a_router`, `agent_endpoints_router`, `anthropic_router`,
   `anthropic_skills_router`, `claude_code_marketplace_router`,
   `guardrails_router`, `compliance_router`, `config_override_router`,
   `jwt_key_mapping_router`, `mcp_management_router`, `policy_router`,
   `scim_router`, etc. The `attach_lazy_features(app)` call at the
   bottom of FastAPI app construction is removed and replaced with the
   pre-#26534 `app.include_router(...)` chain.

3. **Test cleanup** in `tests/proxy_unit_tests/test_proxy_routes.py:39-53`:
   the force-load shim that iterated `LAZY_FEATURES` and called
   `feat.register_fn(app, module)` so `test_routes_on_litellm_proxy()`
   could see the full route set is removed — eager imports now make the
   shim unnecessary.

4. **Test deletion** in `tests/test_litellm/proxy/test_proxy_server.py`
   (lines 5471-5725): the entire `TestLazyFeatureRegistry`,
   `TestLazyFeaturesNotImportedAtStartup`, and `TestLazyFeatureMiddleware`
   classes are removed (~250 lines), including the subprocess-isolated
   `test_heavy_modules_absent_at_startup` that asserted the deferred
   modules were *not* in `sys.modules` after fresh import — that's the
   exact test that was guarding the property the revert is restoring to
   "no longer guaranteed."

## Why `merge-as-is`

This is a straight revert of a feature whose review (PR #26534, see
`reviews/2026-W17/BerriAI-litellm-pr-26534.md`) had already flagged the
risk-shape of the change: a hand-curated prefix/suffix denylist with
`# Trailing slash to avoid matching /policies/...` style discriminators is
exactly the kind of correctness surface that breaks on quiet route-name
edits in any of 25 modules. Possible reasons for the revert (the PR body
just says "Reverts BerriAI/litellm#26534"):

- **Prefix-collision regression**: the `mcp_app` mount at `/mcp` will
  shadow `mcp_byok_oauth`'s `/v1/mcp/oauth` and `mcp_discoverable`'s
  `/.well-known/oauth-` if the matcher walks the registry in the wrong
  order. Production hits would manifest as a 404 for any OAuth dance
  endpoint that arrived before `/mcp` was warm.
- **OpenAPI-schema gap**: `/openapi.json` omitting routes until each
  feature is warmed breaks any client/SDK generation pipeline that
  assumes the schema is complete on startup. Internal SDK regeneration
  in CI would have caught this.
- **First-request-latency regression**: the original PR called out
  "1-3 s for heavy modules" on first hit, but production ASGI traffic
  routinely terminates at p99-budget gateways with sub-second deadlines;
  a 1-3 s tax on the *first* `/v1/agents` call after a deploy would
  trigger user-visible 504s.

Whichever cause(s) drove the revert, the right shape is exactly this: rip
out `_lazy_features.py`, restore eager imports, drop the now-orphaned
test classes that pinned the deferred-import property, and drop the
test-side force-load shim that was a workaround for the same property.
The diff is mechanical and easy to verify by reading: every removed
import in the lazy module appears as a restored top-level import.

## Nits (non-blocking)

- **Memory-savings claim should be retired in docs**. If the original
  PR had a marketing-side mention of "~700 MB idle savings," any release
  notes / CHANGELOG entry referencing #26534 should be retracted in the
  same release as #26727 — otherwise operators reading historical notes
  will keep asking why their idle RSS hasn't dropped.

- **No replacement strategy is captured**. A post-mortem-style comment
  in the revert PR body (or a follow-up issue) would help: "we want
  the idle-memory savings, but the right shape is X — e.g. an opt-in
  `--minimal-features` flag that fails fast on a request to a
  not-loaded prefix, rather than a transparent middleware." Without
  that capture, the next attempt will rediscover the same prefix-
  collision class of bug.

- **Test deletion vs test repurposing**: `test_heavy_modules_absent_at_startup`
  was a useful negative-shape assertion — it could be inverted into a
  *positive* "all advertised routers ARE imported at startup" test in
  the same revert, pinning the property the revert is restoring. Worth
  doing in a follow-up commit; not worth blocking the revert on it.

## What I learned

When a "lazy-load N modules behind a prefix-match middleware" change has
to be reverted, the cleanest revert pattern is exactly what this PR does:
delete the new module wholesale, restore the eager imports as a single
hunk in the top-level orchestrator, and *also* delete the negative tests
that pinned the deferred-import property — leaving them in place would
have inverted-failed in CI on the next push and forced a follow-up.
The harder question is whether the *motivation* for #26534 is still
valid (saving idle RSS for deployments that disable optional features),
and the answer is "yes, but the right primitive is an explicit opt-in
config flag with a fail-fast failure mode, not a transparent matcher."
A revert is the right shape for now; the right re-attempt later is a
different shape than the one being reverted.
