# BerriAI/litellm #27014 — fix(proxy): scope team and agent activity endpoints per-entity (VERIA-43)

- **Repo:** BerriAI/litellm
- **PR:** https://github.com/BerriAI/litellm/pull/27014
- **HEAD SHA:** `38fd1a9`
- **Author:** stuxf
- **Verdict:** `merge-after-nits`

## What the diff does

Closes two cross-tenant data-disclosure bugs in the management activity endpoints:

**(1) `/team/daily/activity` — `team_endpoints.py:5026-5052`.** Prior loop was "if caller is admin OR has `/team/daily/activity` permission on *any* one of the requested teams, set `has_full_team_view=True` and break." So an admin of `team-A` who requested `team_ids=team-A,team-B` got the full unfiltered breakdown for *both* teams, including per-API-key spend rows for `team-B`. Fix inverts the polarity: start with `has_full_team_view=True`, iterate every requested team, and set `False`+break the moment any team fails the (admin OR permission) check. Net effect: full view requires the predicate to hold *for every requested team*; otherwise the whole response degrades to "filtered by caller's own API keys" (which they can already see). Comment at `:5028-5034` correctly names the prior bug.

**(2) `/agent/daily/activity` — `agent_endpoints/endpoints.py:976-1032`.** Previous code passed `where_condition = {}` for any non-admin caller, so an empty `agent_ids` query returned every agent's spend rows on the proxy. Fix adds a non-admin scoping branch:

- Compute `permitted_agent_ids = await AgentRequestHandler.get_allowed_agents(...)`.
- If empty, fall back to agents the caller created (`created_by=user_id`). The **load-bearing guard** at `:996-998` is `if user_api_key_dict.user_id is None: permitted_agent_ids = []` — without it, a literal `None` written into Prisma `where={"created_by": None}` resolves to `created_by IS NULL` and would expose every ownerless agent's rows.
- Intersect caller-supplied `agent_ids` with `permitted_agent_ids` (don't trust the request).
- If the intersection is empty, return an empty `SpendAnalyticsPaginatedResponse` *without* querying — closes the "scan everything" timing channel too.

## Why it's right

- **Fail-closed default**: in both endpoints the new shape says "you see what you have explicit rights to see; the absence of explicit rights means empty, not everything." That's the only correct disposition for analytics endpoints.
- **The Prisma `created_by IS NULL` pitfall is named explicitly** — `test_agent_activity_keyless_caller_does_not_query_created_by_null` (`test_activity_tenant_scoping.py:236-281`) is the dispositive test that catches the foot-gun: a service-account key with no `user_id` triggers neither the owned-agents query nor the activity query, the response is empty, and `find_many` is asserted not_called.
- **Test coverage is comprehensive on the `team` side**: `test_team_activity_requires_admin_on_every_requested_team` proves admin-of-A + member-of-B forces the api_key filter; `test_team_activity_full_view_when_admin_of_all_requested_teams` is the matching positive that locks "we didn't over-restrict."
- **Test coverage on the `agent` side** covers the four state combinations: admin-unscoped, non-admin-no-perms-falls-back-to-owned, non-admin-intersects-explicit-agent-ids, non-admin-no-access-empty-page — the four cells of the truth table.
- **Comments name the prior bug shape** (`:5028-5034` for team, `:976-988` for agent) — load-bearing because a future "let's restore the OR-any-team behavior for UX" change should bounce off the comment.

## Nits

1. **`get_allowed_agents` returning `[]` is overloaded** (means both "no restrictions configured, can see everything" *and* "explicitly restricted to nothing"). The fallback-to-owned path conflates these; an explicit "restricted to nothing" caller now sees their owned agents instead of zero. If `get_allowed_agents` could return `None` to mean "unrestricted" vs `[]` for "explicitly empty," the semantics would be clean. Worth a follow-up TODO.
2. **`owned_records = await prisma_client.db.litellm_agentstable.find_many(where={"created_by": user_api_key_dict.user_id})`** at `:1003-1006` returns full rows just to extract `agent_id`. Add `select={"agent_id": True}` projection — keeps memory footprint small for tenants with many agents.
3. **The team-activity fallback "they can re-request the admin-only teams separately to get the wider view"** (comment at `:5031-5034`) is the right behavior for safety but a poor UX. Worth a 403 with explanatory body instead of a silent degradation, or at minimum a `metadata.partial_view: true` flag in the response so callers can detect the degradation.
4. **No test for "admin of every team but NOT proxy admin"** is the single most likely edge to regress; `test_team_activity_full_view_when_admin_of_all_requested_teams` covers the team-admin path but not "team admin with permission-but-no-admin on one team," which exercises the `is_admin or has_perm` short-circuit.
5. **`DailySpendMetadata(...)`** literal at `:1018-1031` is hand-built — extract a `DailySpendMetadata.empty(page=page)` constructor so the empty-page shape is one place; otherwise the `total_pages=0, has_more=False` invariants drift across endpoints.
6. **Two CVEs in one PR** (the team-activity over-grant and the agent-activity unscoped) — splitting would make downstream backports cleaner.

`merge-after-nits` — right diagnosis, right load-bearing guards (the `created_by IS NULL` pitfall named at `:996-998` is the kind of detail that proves the author understood the real failure mode), comprehensive truth-table tests, but the empty-vs-unrestricted ambiguity and the response-projection nits want follow-ups.
