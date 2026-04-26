---
pr: 26523
repo: BerriAI/litellm
sha: 99b731a3d5969fa2b21c13280e848b57c4fa0503
verdict: needs-discussion
date: 2026-04-26
---

# BerriAI/litellm #26523 — feat(teams): per-model team member budgets (clean reimplementation)

- **Author**: ishaan-berri
- **Head SHA**: 99b731a3d5969fa2b21c13280e848b57c4fa0503
- **Link**: https://github.com/BerriAI/litellm/pull/26523
- **Size**: ~2500 diff lines across schema (`schema.prisma`), 2 new Prisma migrations, `proxy/_types.py`, `proxy/auth/{auth_checks,user_api_key_auth}.py`, `proxy/common_utils/reset_budget_job.py`, `proxy/db/{db_spend_update_writer,prisma_client}.py`, `proxy/db/db_transaction_queue/{redis_update_buffer,spend_update_queue}.py`, `proxy/hooks/proxy_track_cost_callback.py`, plus model-prices JSON.

## Scope

Reimplementation of the previously-shipped per-model team-member budgets feature (#26471 was the first cut, this PR replaces it). The headline architectural change is moving from a JSON `model_spend` column on `LiteLLM_TeamMembership` to a dedicated `LiteLLM_TeamMemberModelSpend` table keyed by `(user_id, team_id, model)` so per-model spend updates are atomic increments instead of read-modify-write on a JSON blob — which the migration comment explicitly calls out as raced under concurrent pod flushes.

## Specific findings

- `schema.prisma:613-633` — old `LiteLLM_TeamMembership` is now formatting-cleaned (column alignment) and a new `LiteLLM_TeamMemberModelSpend` table is added with composite PK `(user_id, team_id, model)` and a `Float spend @default(0.0)`. The composite PK is correct and supports `INSERT … ON CONFLICT … DO UPDATE SET spend = spend + EXCLUDED.spend` for atomic upsert-increment.
- `migrations/20260425000000_add_team_membership_model_spend/migration.sql` — **no-op migration** with a comment explaining the column was superseded. This is unusual: typically you'd just not create the migration. The comment claims "JSON approach was dropped" but no `DROP COLUMN` is here — meaning if a previous deployment ran on this branch with #26471 and got the JSON column, it stays. Probably fine (no consumers read it once #26471 is fully removed) but should be confirmed and the no-op rationale should appear in the PR body, not just the migration file.
- `migrations/20260425010000_add_team_member_model_spend_table/migration.sql:1-8` — clean `CREATE TABLE IF NOT EXISTS` with the composite PK. Idempotent. Good.
- `litellm/caching/in_memory_cache.py:165-170` — adds an `nx` (set-if-not-exists) kwarg to `set_cache`. Implementation: if `nx=True` and key exists, call `evict_element_if_expired` and only proceed if the element was actually evicted. This is a behavioral change to the in-memory cache shared by everything, not just the team-budget code, and there's no test for the `nx` semantics in the diff scope I reviewed. Also, `evict_element_if_expired` is presumably idempotent on already-expired entries; the `if not ...: return` branch swallows the case where the key exists and is unexpired without surfacing it (no return value, no log). Callers can't distinguish "set" from "skipped". Consider returning a bool.
- The `model_prices_and_context_window_backup.json` change adds an `azure/gpt-5.5` entry with `1050000` max_input_tokens. Out of scope for the PR title; should be split into its own PR ("data: add azure/gpt-5.5 pricing") so the team-budget review stays focused.
- I did not exhaustively re-read the auth_checks / user_api_key_auth / reset_budget_job / spend_update_queue / redis_update_buffer changes (the diff is ~2500 lines and the PR body labels this as a "clean reimplementation" of an already-reviewed feature — I'm calling out architectural concerns rather than line-by-line auditing the spend math). Reviewers familiar with #26471 should diff against that PR rather than against main.

## Risk

High due to scope and shape. The architectural improvement (atomic table over JSON) is the right call — RMW on JSON spend columns is a textbook race. But:
1. The PR bundles unrelated changes (azure/gpt-5.5 pricing entry, an `nx` kwarg added to the global in-memory cache that any caller could trip).
2. The "no-op" migration paired with a real migration is a bisect/rollback hazard — if the no-op runs but the real migration fails, you have an inconsistent migration history with no clean recovery path. Either drop the no-op or merge them into one transactional migration.
3. Replacing an already-shipped feature (#26471) means there are deployments running with the JSON-column shape. The PR has no explicit "if you're on #26471, here's the data migration" path. If anyone went to prod on #26471, their accumulated `model_spend` JSON values will be lost.
4. The `nx` kwarg silent-skip semantics (no return, no log) need a contract decision before this lands.

## Verdict

**needs-discussion** — the underlying architectural pivot is correct and welcome, but this PR mixes a schema migration, a global-cache contract change, an unrelated pricing-data addition, and a replacement of a feature that has already shipped. Recommend splitting into:
1. `data: add azure/gpt-5.5 pricing entry` (1-line JSON change)
2. `feat(cache): add nx semantics to InMemoryCache.set_cache` (with test + return value)
3. `feat(teams): per-model team-member budgets via dedicated table` (the actual reimplementation, ideally with a documented migration path from #26471's JSON shape if anyone deployed it)

The maintainers may want to keep the bundled shape for shipping velocity — that's their call — but the migration-history concern (#2) and the deployment-data-loss concern (#3) need explicit answers in the PR body before merge.
