# Review: BerriAI/litellm#26471 — feat(teams): per-model team member budgets

- **PR**: https://github.com/BerriAI/litellm/pull/26471
- **State**: OPEN
- **Author**: ishaan-berri
- **Range**: +465 / −27
- **Head SHA**: `d9067925f5d39a79e76a8f5ac1cc7b8250912eeb`
- **Base SHA**: `70492cee4282541256fb9ac963be94412b1a109c`
- **Verdict**: request-changes
- **Reviewer date**: 2026-04-25

## What the PR does

Adds per-(team, member, model) budget enforcement. Introduces a new
`LiteLLM_TeamMemberModelSpend` table (composite PK
`(user_id, team_id, model)`), a `team_member_model_max_budget` field on
`NewTeamRequest` / `UpdateTeamRequest`, an
`_check_team_member_model_budget` auth check in
`litellm/proxy/auth/auth_checks.py:650+`, queue plumbing in
`db/db_spend_update_writer.py`, and a `_commit_team_member_model_spend_to_db`
upsert path. The enforcement uses an in-memory counter cache
(`spend:team_member:{user_id}:{team_id}:model:{model}`) seeded from the DB
on cache-cold via a `-1.0` sentinel.

## Strengths

- The migration story is honest: the first migration is a no-op stub
  documenting why the JSON-on-`LiteLLM_TeamMembership` approach was
  abandoned (read-modify-write race under concurrent pod flushes), and the
  second creates a dedicated table designed for atomic upsert+increment.
  Good engineering hygiene to leave that paper trail.
- `_check_team_member_model_budget` correctly distinguishes "cache cold"
  (`-1.0` sentinel) from "real zero balance" and uses `async_set_cache`
  rather than `async_increment_cache` on cold-seed — the comment on
  `auth_checks.py:~3380` explicitly calls out the double-INCRBYFLOAT
  duplication risk if two requests race a miss. Correct fix.

## Blocking concerns

1. **`team_member_model_max_budget` is read from `team.metadata`, not the
   typed column.** `_check_team_member_model_budget` does
   `team_object.metadata.get("team_member_model_max_budget", {})`, but
   `NewTeamRequest` exposes it as a top-level field. Where does the typed
   field land in the DB? If it's serialized into `metadata` by the team-
   create path, that's worth a comment + a test. If it's stored in a typed
   column that this checker isn't reading, the feature silently no-ops.
2. **Counter-key drift between writer and reader.** The auth checker keys
   the cache as
   `spend:team_member:{user_id}:{team_id}:model:{model}`, but the writer in
   `db_spend_update_writer.py:~580` enqueues with
   `team_id::{team_id}::user_id::{user_id}::model::{model}` and then
   `_commit_team_member_model_spend_to_db` parses *that* shape. I don't
   see the bridge that mirrors writer-side increments back into the
   `spend:team_member:...:model:...` cache the reader checks. Without it,
   spend grows in the DB but the auth check keeps reading stale cache (or
   re-seeding from DB on every cold key), and the 429 fires only after the
   cache TTL expires. Please add a test that drives N requests through the
   full request → cost-callback → next-auth-check loop and asserts the
   429 fires at request N+1, not whenever the TTL happens to flip.
3. **`model_group` vs `model` ambiguity.** `_update_team_db` is called
   with `model=payload_copy.get("model_group") or payload_copy.get("model")`.
   If the user sets a budget keyed on `gpt-4o-mini` but the routing layer
   resolves to a deployment named `azure-gpt-4o-mini-eastus`, the wallet
   debits one key and the budget check reads another. Document the
   contract: budgets are keyed on the *requested* model name (the
   `model_group`), and add a regression test for an aliased deployment.
4. **No premium gate visible in this diff** for what looks like an
   enterprise feature; the project usually fences these. Confirm.

## Nits

- The migration filename uses the future date `20260425000000` — fine for
  this calendar but worth confirming the migration runner sorts
  lexicographically and not via a parsed timestamp.
- Spelling: "superseded" ✓ (good — common typo).
- `Litellm_EntityType.TEAM_MEMBER_MODEL` — the rest of the enum is
  SCREAMING_SNAKE; this matches. ✓

## Recommendation

Hold for changes. The data model and migration discipline are solid, but
the writer/reader cache-key mismatch (concern #2) looks like it would
prevent the feature from working under normal request flow, and the
metadata-vs-field plumbing (concern #1) needs a smoke test before merging.
