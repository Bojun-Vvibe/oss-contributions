# PR #26390 — guardrail param masking + team-scoped list endpoint

**Repo:** BerriAI/litellm • **Author:** yuneng-berri • **Status:**
open • **Net:** +65 / −31

## Change

Three related fixes in the guardrails surface:

1. `_get_masked_values` (in `litellm_core_utils/litellm_logging.py`)
   refactored to recurse into nested dicts and to recognize three
   additional sensitive keyword patterns: `credentials`, `password`,
   `passwd`. Logic extracted into an inner `_mask_value` helper.
2. `_row_to_submission_item` in `guardrail_endpoints.py` (used by
   `/guardrails/submissions`) now applies `_get_masked_values` to
   `litellm_params` before returning. Previously the raw dict was
   leaked.
3. `list_guardrails_v2` (`/v2/guardrails/list`) now requires a
   `UserAPIKeyAuth` parameter and, for non-admin callers, restricts
   the result to guardrails that are either un-owned (`team_id is
   None`) or owned by one of the caller's teams. Excluded DB IDs
   are pre-seeded into `seen_guardrail_ids` so an in-memory
   guardrail with the same ID can't sneak the row back in.
4. `approve_guardrail_submission` propagates `team_id` into the
   in-memory dict so ownership is preserved post-approval.

## Why this is three fixes in one PR

That packaging is a smell — each of (1), (2), (3) could ship
independently and they have different blast radii:

- (1) is a strict improvement to a shared utility; no caller can
  regress from a recursive mask.
- (2) is a confidentiality leak fix.
- (3) is an authorization model change with API-shape implications
  (the route now requires auth, where the old `Depends` block was
  attached to the route decorator — same effective auth, but the
  function signature changed).

If (3) regresses, the team-scoped filter could over-exclude and
admins might miss guardrails. Splitting (3) into its own PR with a
focused changelog entry would give operators a clear rollback
target.

## Risk in the masking refactor

The new recursive `_mask_value` is invoked **only when the parent
key matches** a sensitive keyword. So a dict like:

```python
{
  "config": {
    "api_key": "sk-secret",
    "endpoint": "https://..."
  }
}
```

— will leak `api_key` because the parent key `config` is not
sensitive. The recursion only kicks in when the outer key is
sensitive (e.g. `credentials = {...}`), which is the documented
"nested credentials object" case. Users who model their config as
top-level non-sensitive keys with sensitive sub-keys will continue
to leak. Worth a comment in the docstring saying recursion only
fires on a sensitive parent key.

A safer (but more expensive) alternative: walk every dict
unconditionally and apply the per-key check at every level. The
current code is `O(n_top_level_keys)`; the safer version is
`O(n_total_keys)` but resistant to the schema-shape pitfall above.

## Risk in the team-scoping logic

`is_admin = user_api_key_dict.user_role == LitellmUserRoles.PROXY_ADMIN`
is a single role check. There's a third tier in litellm — internal
users with cross-team visibility — that may be incorrectly treated
as "non-admin" here and have their list under-populated. Verify
against `LitellmUserRoles.INTERNAL_USER` and similar constants in
the same enum.

The pre-seeding pattern is clever:

```python
seen_guardrail_ids: set = excluded_guardrail_ids.copy()
```

— this prevents in-memory guardrails with a colliding ID from
backdoor-restoring an excluded DB guardrail. Good. But it changes
the **identity policy**: previously `seen` was a dedup-only set,
now it's also an exclusion set. If two guardrails legitimately
share an ID across DB and in-memory (which the prior dedup loop
implies is possible), the in-memory one is now silently dropped
even when the user *should* have access (e.g., a config-yaml
guardrail with a name collision against an excluded team
guardrail). Worth a test for that edge case.

## Sharp edge: route signature change

The dependency moved from:

```python
dependencies=[Depends(user_api_key_auth)],
async def list_guardrails_v2():
```

to:

```python
async def list_guardrails_v2(
    user_api_key_dict: UserAPIKeyAuth = Depends(user_api_key_auth),
):
```

Both forms enforce auth, but only the second exposes the auth
object inside the handler — required for the new team filter.
Test fixtures that called `list_guardrails_v2()` without args now
need to pass the auth dict; the PR updates three tests accordingly.
Any out-of-tree caller (custom plugins, extension modules) that
imported and called this function directly will break. Worth a
mention in the changelog.

## Verdict

Three good fixes packaged into one PR. The masking recursion is
correct but partial; the team-scoping is correct but role-fragile;
the API signature change is necessary but breaking. Splitting and
landing in three PRs would lower review risk and give operators
clearer rollback granularity. As one PR, it's still a net positive
— the confidentiality leak (2) alone justifies merge.

## What I learned

When a refactor combines a "strict improvement" (better masking)
with an "authorization model change" (team scoping), the
authorization change should be the gating concern in review. It's
the only one that can over-restrict and cause user-visible "I
can't see my guardrail" tickets.
