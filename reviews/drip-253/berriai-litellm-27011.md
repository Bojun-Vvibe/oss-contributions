# BerriAI/litellm #27011 — fix(proxy): close project hijacking and key org IDOR (VERIA-55)

- **Repo:** BerriAI/litellm
- **PR:** https://github.com/BerriAI/litellm/pull/27011
- **HEAD SHA:** `1b2756811e0118e4df1bec0b665f4a6003a3c5ad`
- **Author:** stuxf
- **Verdict:** `merge-after-nits`

## What the diff does

Two thematically-related authorization fixes:

**1. Project hijacking via `team_id` body parameter**

`enterprise/litellm_enterprise/proxy/management_endpoints/project_endpoints.py:588-630`
— prior to the fix, `update_project` resolved
`team_id_to_check = data.team_id or existing_project.team_id` and
passed the resulting `team_object` into the permission check.
A team admin of any team could pass their own `team_id` in the request
body, the permission check would then validate "is this user an admin
of the team they supplied?", and the answer was always yes —
hijacking the project into their own team's namespace.

The fix removes the `team_object` injection from the permission check
entirely (`:606-609`) so `_check_user_permission_for_project` falls
back to its safe default of looking up the project's *current* team.
A separate post-permission gate at `:614-635` then checks "if the
caller is *also* trying to reassign to a different team, do they have
admin rights on the destination team too?" via a second
`_check_user_permission_for_project` call against `data.team_id`.
The `target_team_obj` lookup is preserved for the limit-validation
step at `:636-641`.

**2. Key-update org IDOR**

`litellm/proxy/management_endpoints/key_management_endpoints.py:1171-1207,2204-2226`
— prior to the fix, `/key/update` accepted a body
`organization_id` and applied it without verifying the caller is a
member of that organization. Any caller could point their own key at
an arbitrary `organization_id` and gain whatever org-scoped quotas /
permissions / spend pool that org carried.

New `_validate_caller_can_assign_key_org` (`:1171-1206`) uses the
same membership-rule shape as `validate_key_list_check`: looks up the
caller's user row + `organization_memberships`, builds the set of
`member_org_ids`, raises 403 if `organization_id` is not in the set.
Caller is invoked from `_validate_update_key_data` at `:2218-2226`
gated on (a) `data.organization_id is not None`, (b) value differs
from the existing key's `organization_id`, (c) caller is not a proxy
admin (admins bypass).

Tests at
`tests/test_litellm/proxy/management_endpoints/test_project_org_authz.py`
(+195 lines) cover six cases: project-perm uses-current-team-not-supplied
(`:42-66`), team-admin-of-existing-team-allowed (`:69-87`),
proxy-admin-always-allowed (`:90-109`), key-org-allows-member
(`:138-156`), key-org-blocks-non-member (`:159-180`),
plus `find_unique` mock plumbing.

## Why it's right

The project fix is the canonical "permission check evaluated against
caller-supplied identity is no permission check at all" pattern. The
right primitive is "permission check evaluated against the resource's
*current* state" (i.e. the project's existing `team_id`), and the
diff achieves this by *removing* the `team_object` injection so the
inner `_check_user_permission_for_project` falls back to looking up
the team itself from the existing project's team_id. That removal is
load-bearing — the helper *would* have used `team_object` if passed,
even though it normally hits the DB for the existing-team lookup.

The two-gate design (permission to *edit* the project at the current
team + permission to *reassign* to the new team) is correct
defense-in-depth: a team admin shouldn't be able to dump projects
into another team's namespace any more than a non-admin can claim
projects from another team. The dispositive comment at `:603-605`
("Sourcing the team from `data.team_id` would let an admin of any
team pass the check by supplying their own team_id, hijacking the
project") names the failure mode in plain English at the call site,
which is the right place to land it.

The key-org fix mirrors the existing membership rule from
`validate_key_list_check` — the docstring at `:1175-1180` explicitly
names the parallel ("Mirrors the org-membership rule already enforced
on `/key/list` in `validate_key_list_check`"), which is the right
forcing function: if the read-side IDOR was closed by membership
check, the write-side IDOR closes the same way. The proxy-admin
bypass at `:2216` (`not _is_proxy_admin`) is correct and matches
sibling org-related authorization shapes.

The `data.organization_id != _existing_org_id` gate at `:2215` is the
right narrowing — re-passing the same org_id should not require a
fresh membership round-trip (it would be a no-op anyway).

## Nits

1. **No test for the "reassign to different team" gate** at
   `project_endpoints.py:614-635`. The test file covers
   `_check_user_permission_for_project` in isolation but not the
   end-to-end `update_project` path where the second permission
   check fires. A test posting `{team_id: "team-B"}` as a team-A
   admin and asserting 403 would lock the most important
   architectural property of the fix (the second gate exists and
   fires).

2. **`_validate_caller_can_assign_key_org` raises 403 when
   `user_api_key_dict.user_id is None`** (`:1184-1188`). For
   service-account keys (which may legitimately have no `user_id`)
   this is a behavior change worth flagging in the PR body —
   service-account flows that previously could update a key's org
   now get a 403. If service accounts shouldn't be able to do this
   at all, the message should say so; if they should via a
   different gate (e.g. team membership), the helper needs a
   second arm.

3. **`organization_memberships` access via `getattr` with `None`
   default** at `:1191-1194` is defensive against a Prisma include
   that silently returns nothing, but the same shape would mask a
   legitimate Prisma error (e.g. include schema mismatch). A
   `try/except` with explicit logging would surface failures
   without converting them to "no memberships".

4. **`data.organization_id != _existing_org_id`** at `:2215`
   compares two values that may both be `None` — `None != None` is
   `False` in Python, which is what we want (no-op), but worth a
   comment locking the intent. If the existing key has
   `organization_id=None` and the caller passes `organization_id=""`
   (empty string), the comparison fires and the membership check
   runs against an empty string, which `_validate_caller_can_assign_key_org`
   will likely 403 on (no member of org_id="" exists). Defensive
   `data.organization_id and ...` would be more robust.

5. **The two fixes are conceptually independent** (project-perm vs.
   key-org IDOR) and could ship as two PRs for cleaner audit
   traceability — one CVE per PR is easier for downstream
   security tracking. Not a blocker but worth noting since the
   VERIA-55 ID is shared across both.

6. **`raise HTTPException(status_code=403, detail=f"Caller is not a
   member of organization_id={organization_id}")`** at `:1203` leaks
   the organization_id back to the caller via the error message.
   This is fine when the org_id is something the caller already
   knows (they passed it in), but if the policy is "don't confirm
   or deny the existence of an org you're not a member of", the
   error message should be `"Insufficient permissions to assign
   key to that organization"` without echoing the id.

## Verdict rationale

Right diagnosis (two distinct IDOR shapes both keyed on caller-supplied
identity bypassing membership checks), right fix shape (remove the
caller-supplied `team_object` injection so the helper falls back to
safe lookup; mirror the read-side membership rule for the write-side
gate), right test surface for the helpers in isolation. Missing
end-to-end test on the second project gate, service-account behavior
change worth flagging, and the two fixes could ship separately for
audit clarity.

`merge-after-nits`
