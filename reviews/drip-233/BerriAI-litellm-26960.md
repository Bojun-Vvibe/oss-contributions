# BerriAI/litellm#26960 — feat(mcp): enforce org-level MCP server and toolset permissions

- **Author:** Sameer Kankute (Sameerlite)
- **Head SHA:** `03f6818a547b984312ea6563866672bed1c7ce53`
- **Base:** `litellm_internal_staging`
- **Size:** +418 / -3 across 2 files
- **Files changed:** `litellm/proxy/_experimental/mcp_server/auth/user_api_key_auth_mcp.py`, `tests/test_litellm/proxy/_experimental/mcp_server/auth/test_user_api_key_auth_mcp.py`

## Summary

Adds organization tier to the MCP-server-and-toolset permission hierarchy. The prior model intersected `key ∩ team ∩ end_user ∩ agent`; this PR adds an org-level "ceiling" rule applied at the end: if the org has an explicit `mcp_servers` list, the combined lower-tier result is capped to it; if the org has no list, no extra restriction. Includes a cache-keyed lookup with a sentinel value so orgs *without* MCP permissions configured (the common default) don't pay a DB round-trip per request.

## Specific code references

- `user_api_key_auth_mcp.py:411-416`: docstring rewrite of the permission hierarchy from a 4-step model to a 5-step model with explicit "org acts as a ceiling" semantics. The wording "If the org has no list, no extra restriction is applied" correctly distinguishes "org has empty list" (which would be a different, deny-all case) from "org has no `object_permission` row at all" (no-op).
- `user_api_key_auth_mcp.py:506-526`: new org-ceiling block in `get_allowed_mcp_servers`. Two-arm logic: if `len(allowed_mcp_servers) > 0` (lower tiers added explicit restrictions), intersect with `allowed_mcp_servers_for_org`; else (`else: allowed_mcp_servers = allowed_mcp_servers_for_org`), the org list becomes the entire permitted set. Correct implementation of the docstring's ceiling semantics — the `else` branch is load-bearing because without it, an org-only restriction would be no-op when no other tier had restricted the list.
- `user_api_key_auth_mcp.py:669-689`: parallel org-ceiling treatment in `get_allowed_tools_for_server`. Same two-arm pattern (`set(allowed_tools) & set(org_tools)` vs `list(org_tools)`). Correctly uses `expand_tool_permissions(org_obj_perm.mcp_tool_permissions).get(server_id)` so the per-server tool ceiling only applies to tool permissions for *this specific server*. Comment at `:670-672` notes "_get_org_object_permission uses user_api_key_cache, so this is not a fresh DB round-trip when get_allowed_mcp_servers was already called" — correct micro-perf observation worth keeping.
- `user_api_key_auth_mcp.py:856-858`: new sentinel `_ORG_NO_PERMISSION_SENTINEL = "__org_no_mcp_permission__"`. Cache stores this string when DB confirms no `object_permission` row, so subsequent calls return None without re-querying. The string-as-sentinel pattern (vs a dedicated typed marker) is acceptable here because the cache is untyped; the namespace prefix `__org_no_mcp_permission__` is collision-resistant in practice.
- `user_api_key_auth_mcp.py:861-907`: new `_get_org_object_permission` helper. Three-step flow: cache get → DB lookup with `include={"object_permission": True}` → cache set (either real `obj_perm` or the sentinel). The `if cached == _ORG_NO_PERMISSION_SENTINEL: return None` branch at `:884-886` is the load-bearing perf win — without it, a per-request DB call would occur for every org that hasn't configured MCP permissions (likely the vast majority).
- `user_api_key_auth_mcp.py:910-955`: new `_get_allowed_mcp_servers_for_org` helper, structurally parallel to the existing `_get_allowed_mcp_servers_for_team`. Combines three sources (`expand_permission_list(mcp_servers)` direct, `_get_mcp_servers_from_access_groups(mcp_access_groups)` via groups, and `expand_tool_permissions(mcp_tool_permissions).keys()` implicit-via-tool-perms) into one deduplicated list via `list(set(all_servers))`. The deduplication is correct because the three sources can overlap and downstream callers don't need ordering.
- `test_user_api_key_auth_mcp.py:2143+`: new "Org-level MCP permission tests" section (only header visible in the diff window — the +418 size implies substantial test additions). Verify these include: (a) org with no permission row → no restriction, (b) org with empty list → empty intersection, (c) org with list intersecting non-trivially with team/key restrictions, (d) sentinel cache hit avoids DB call (mock prisma to verify zero calls on second invocation), (e) parallel coverage for `get_allowed_tools_for_server`'s new org branch.

## Reasoning

Clean architectural addition that follows the existing `_get_allowed_mcp_servers_for_{team,end_user,agent}` pattern symmetrically — same method signature shape, same `try/except` defensive wrapper, same `verbose_logger.warning` on failure with `return []` fallback. The cache sentinel pattern is a thoughtful perf-aware addition that the prior tiers don't bother with — worth backporting to the team/end_user paths in a follow-up if those also see the "tier not configured" common case.

The "ceiling" semantics (intersection-when-both-non-empty, ceiling-replaces-when-lower-empty) is the correct policy choice for org tier specifically, because orgs are the highest tier and "I haven't configured MCP at the org level but I want to allow keys to use what they're individually permitted" needs to be a no-op rather than a deny-all. The two-arm `if/else` at `:514-525` is the load-bearing implementation of this distinction.

Concerns:
- **Cache invalidation story.** The sentinel and the real `object_permission` are both cached indefinitely (no `ttl=` on `async_set_cache`). If an admin mutates an org's `object_permission`, the cache won't reflect it until the cache evicts. Verify the existing org-update path invalidates `f"org_object_permission:{org_id}"`, or add a TTL.
- **Tool-permission-implies-server-allowed semantics.** At `:947-951` `tool_perm_servers = list(expand_tool_permissions(...).keys())` adds servers to the allowed-servers list purely because they have tool entries in the org's `mcp_tool_permissions`. This is the right call (granting tool perms on a server implicitly grants the server) and matches the team-tier behavior — confirm this in the PR body so the maintainer doesn't second-guess.
- **`org_id` plumbing.** `get_allowed_tools_for_server` at `:668` reads `user_api_key_auth.org_id` directly without the `if user_api_key_auth and ...` guard used at `:506` in `get_allowed_mcp_servers`. If `user_api_key_auth` is None at this site (which the code path elsewhere does check), this would AttributeError. Trace the callers to confirm `user_api_key_auth` is always non-None here, or add the guard for symmetry.

## Verdict

**merge-after-nits**
