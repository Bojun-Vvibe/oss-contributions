# BerriAI/litellm#26889 — Fix org-level MCP permission enforcement

- PR: https://github.com/BerriAI/litellm/pull/26889
- Head SHA: `feda4fa70806404c0cc47017d57911bb629cd23a`
- Author: Genmin (Joey Roth)
- Files: `proxy/_experimental/mcp_server/auth/user_api_key_auth_mcp.py`
  +160/−4, `tests/.../test_user_api_key_auth_mcp.py` +374/0; +534/−4
- Closes #26791

## Context

The proxy UI/API let operators configure organization-level MCP
servers and tool permissions, but `MCPRequestHandler
.get_allowed_mcp_servers()` only evaluated four levels: key, team,
end_user, agent. Result: a key/team scoped under an organization
inherited *nothing* from the org's MCP allowlist and was bounded by
*nothing* on the org side either. Documented config + silent
runtime divergence — a real authorization gap, not a "your config
won't load" gap.

## Design

The PR threads an `_get_allowed_mcp_servers_for_org()` resolver
into the existing precedence pipeline at
`user_api_key_auth_mcp.py:430-470` and applies it as inheritance
when key/team have no MCP permissions, intersection when they do:

```python
allowed_mcp_servers_for_org = await MCPRequestHandler\
    ._get_allowed_mcp_servers_for_org(user_api_key_auth)
...
if len(allowed_mcp_servers_for_org) > 0:
    if len(allowed_mcp_servers) > 0:
        allowed_mcp_servers = [
            s for s in allowed_mcp_servers
            if s in allowed_mcp_servers_for_org
        ]
    else:
        allowed_mcp_servers = allowed_mcp_servers_for_org
```

Org id resolution at `:580-647` (`_get_org_id_for_user_auth`)
prefers `user_api_key_auth.org_id` (key-direct or JWT-claimed)
and falls back to walking through `team_id → get_team_object →
team.organization_id` so teams nested under an org inherit
correctly. Returns `None` (skip org check) if neither path
yields an id — which is the right shape because `len(allowed_
for_org) > 0` is the gate that activates the org branch, so
"no org" naturally collapses to pre-PR behavior.

The 374-line test file is the load-bearing artifact:
- Inheritance path (org allows {A,B}, key allows nothing → result {A,B})
- Intersection path (org {A,B}, key {B,C} → {B})
- Empty intersection (org {A}, key {B} → []) — the "principal
  is bounded but has no overlap" case
- Org via direct `org_id` and org via team-membership both pinned
- end_user-on-top stays an additional intersection (preserved)

## Risks / nits

- `_get_org_id_for_user_auth` only consults `user_api_key_auth
  .org_id` and `team.organization_id`. Doesn't look at end_user
  or agent for an org claim. Almost certainly correct (an end
  user belongs to a customer that belongs to an org via key
  metadata, not directly), but worth a one-line comment naming
  the choice so a future "let end_user override org" PR has a
  reference point.
- The header docstring still says "Permission hierarchy (all
  rules are intersections)" then enumerates the new five-level
  rule with mixed inheritance + intersection semantics. Strict
  reading: levels 1-3 (org → key → team) are inheritance-when-
  empty + intersection-when-set, level 4 (end_user) is always
  intersection-when-set. Worth tightening the docstring at
  `:413-417` to name "inheritance-or-intersection" explicitly,
  because the current wording will mislead a future reader who
  expects pure intersection at every step.
- Object-permission resolution flows through
  `LiteLLM_OrganizationTable` and `LiteLLM_ObjectPermissionTable`
  — both newly imported. Worth confirming the prisma relation
  for `team.litellm_organization_table.object_permission_id` is
  populated in the migration path so legacy DBs don't silently
  return empty allowlists (which under inheritance-when-empty
  would *open up* permissions, not close them).
- `_get_team_object_permission` uses module-level imports of
  `prisma_client`, `proxy_logging_obj`, `user_api_key_cache`
  inside the function body — pre-existing pattern, but if a
  future test wants to mock these, the in-function import will
  fight the mock. Out of scope for this PR.

## Verdict

**`merge-after-nits`**

Real authorization gap (configured org permissions not
enforced), correct fix shape (inheritance-when-empty +
intersection-when-set, mirroring the existing key/team
precedence), 374 lines of test coverage that pin all the
load-bearing cases including empty-intersection and team-
membership-as-org-bridge. Nits are docstring tightening and
a one-line comment justifying the org-id resolution scope —
both worth landing in the same PR.

## What I learned

The most dangerous shape of an "inheritance-when-empty"
rule is when the empty case opens up permissions instead of
closing them. The PR gets this right by gating on
`len(allowed_mcp_servers_for_org) > 0` — if the org has *no*
allowlist (legacy DB row, fresh org, etc.), the org level
contributes nothing, so the pre-PR semantics are preserved.
The bug-shape would be `if org_obj_perm is not None:` which
silently empties the allowlist for any org row that exists
but has empty permissions.
