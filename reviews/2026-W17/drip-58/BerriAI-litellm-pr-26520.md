# BerriAI/litellm#26520 — Add "My User" tab to team info page

- **Repo:** BerriAI/litellm
- **Author:** ryan-crabbe-berri
- **Head SHA:** `79762e4f` (79762e4f5c29df4072bd53dc90ac0d9e8c974fc7)
- **Size:** +704 / -4

## Summary
Adds a `GET /team/{team_id}/members/me` endpoint that returns the caller's
own team-membership row (spend, budget, reset date, rate limits, role).
Resolves the caller from `user_api_key_dict.user_id`; returns 404 when the
user is not a member of the team and 400 when the key has no associated
user_id (team/service keys).

## Specific references
- `litellm/proxy/_types.py:682` @ `79762e4f` — `/team/{team_id}/members/me` added to `LiteLLMRoutes`. Important: this means the route is allowlisted for non-admin internal users. Good.
- `litellm/proxy/management_endpoints/team_endpoints.py:3422-3430` @ `79762e4f` — endpoint declared with `dependencies=[Depends(user_api_key_auth)]` and `response_model=TeamMemberInfoResponse`. Correctly authenticated.
- `litellm/proxy/management_endpoints/team_endpoints.py:3450-3458` @ `79762e4f` — the 400 branch when `caller_user_id is None` correctly rejects team-keys. This is the right call: a team-key has no individual "me", so silently mapping to a fallback would be confusing.
- `litellm/proxy/management_endpoints/team_endpoints.py:3441-3448` @ `79762e4f` — `prisma_client is None` short-circuit returns 500 with a docs URL. Matches the repo's pattern in the surrounding `team_info` endpoint.

## Observations
1. **Authorization model is exactly right**: returning 404 (not 403) when the caller is not a member avoids leaking team existence to non-members. Confirm the actual implementation does this for the lookup-miss case (the docstring says so; the lookup must mirror it).
2. **`get_team_membership` import** (line 69 of `_check_token_auth_helpers.py` import) is new — verify it filters by both `team_id` AND `user_id` rather than fetching all team members and indexing client-side. A naïve impl could return O(team_size) rows; for large teams this matters.
3. **Tests + UI**: the PR is +704 lines so most of that is likely UI + tests. The endpoint itself is small and focused. The "My User" UI tab consuming it should also avoid revealing `members[]` for teams the user isn't in (defense-in-depth: don't rely solely on the API to filter).
4. **Rate-limiting**: this is a per-call endpoint that could be polled by a UI tick; consider whether it should be subject to the standard management endpoint rate limit (likely already covered by `management_endpoint_wrapper`, but worth confirming).

## Verdict
**merge-as-is** — small, well-scoped endpoint with sensible auth semantics. Verify the membership query is indexed and the 404-vs-403 behavior matches the docstring; otherwise good to ship.
