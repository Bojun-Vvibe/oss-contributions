# litellm PR #26418 — fix(proxy): single-team DB fallback when JWT has no team_id

Link: https://github.com/BerriAI/litellm/pull/26418

## What it changes

JWT-authed requests where the token carries no team claim previously
left `team_id` unset on the resulting `UserAPIKeyAuth`, so spend /
metadata never attached to a team — even when the user record in
LiteLLM's DB belonged to *exactly one* team and the inference was
unambiguous. This PR adds a fallback in `auth_builder` (handle_jwt.py):
if `team_id` is still unset after normal JWT/team resolution and
`user_object.teams` has length 1, attempt `get_team_object` and
(if `user_id` set) `get_team_membership` for that single team.

Both DB calls are wrapped in a single bare `except Exception` that
debug-logs and silently skips on any failure (including the
`HTTPException(404)` that `get_team_object` raises when the row is
missing). 0 or ≥2 teams → no fallback.

38 added / 1 deleted in `handle_jwt.py`, plus 298 lines of
parametrized tests covering: single team success, two-team ambiguity,
zero-team, 404 from `get_team_object`, 500 from `get_team_object`,
membership-lookup error.

## Critique

The intent is reasonable — single-team users on JWT auth are an
extremely common config and silently failing to attach `team_id` is a
meaningful spend-attribution gap. Test coverage is genuinely good and
exercises the failure-mode matrix.

That said, this is "infer security context from DB state" and that
class of fix has sharp edges:

1. **The "exactly one team" invariant is operator-mutable.** If a
   user is added to a second team after the JWT was issued, the
   *same JWT* now infers a different security context (no team)
   than before. Operators expect that adding a team is a strict
   widening, not a behavior flip on a previously working request.
   At minimum, this needs to be documented as a deliberate
   behavior; ideally the fallback should be opt-in via a
   `general_settings` flag (e.g.,
   `infer_jwt_team_from_single_team`) so it doesn't activate by
   default for existing deployments.
2. **Bare `except Exception`.** Catching every exception and
   silently skipping makes debugging real DB outages harder — a
   DB connection failure during `get_team_object` will now look
   identical to "user has 2 teams" from the request's
   perspective. The handler should at least distinguish
   `HTTPException(status < 500)` (skip silently) from any other
   exception (log at WARNING, still skip so auth succeeds).
3. **Spend-attribution back-compat.** Existing dashboards /
   queries built on "JWT-no-team requests have NULL team_id"
   will start showing those rows attributed to a team. Worth
   calling out as a behavior change in the PR body so operators
   know to audit their dashboards.

## Suggestions

- Gate the fallback behind a `general_settings` opt-in flag
  defaulting to off; flip to on in a major version.
- Narrow the exception handler: silent on expected 4xx, WARNING
  log on anything else.
- Add the back-compat note to release notes.

Verdict: useful fix, but should not change auth-derived security
context by default. Land behind an opt-in flag.
