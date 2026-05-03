# BerriAI/litellm PR #27057 — Migrate daily-aggregate `date` columns from String to DATE

- **Repo:** BerriAI/litellm
- **PR:** #27057
- **Head SHA:** `4ee434ad6e730a964bdf50b170af175860bd0849`
- **Author:** mateo-berri
- **Title:** Migrate daily-aggregate `date` columns from String to DATE
- **Diff size:** +335 / -61 across 14 files
- **Drip:** drip-294

## Files changed

- `litellm-proxy-extras/litellm_proxy_extras/migrations/20260502182927_convert_daily_spend_date_to_date_type/migration.sql` (+98/-0) — the actual Postgres migration: 8 `DO $$ BEGIN ... ALTER COLUMN "date" TYPE DATE USING "date"::date; END $$` blocks, each guarded by `IF EXISTS (... data_type = 'text')` so re-runs are idempotent. Targets `LiteLLM_DailyUserSpend`, `LiteLLM_DailyOrganizationSpend`, `LiteLLM_DailyEndUserSpend`, `LiteLLM_DailyAgentSpend`, `LiteLLM_DailyTeamSpend`, `LiteLLM_DailyTagSpend`, `LiteLLM_DailyGuardrailMetrics`, `LiteLLM_DailyPolicyMetrics`.
- `litellm-proxy-extras/litellm_proxy_extras/schema.prisma` (+8/-8), `litellm/proxy/schema.prisma` (+8/-8), `schema.prisma` (+8/-8) — three schema files all updated in lockstep: `date String` → `date DateTime @client.Date`. Three copies of the same change is a footgun — easy to drift in future PRs.
- `litellm/proxy/db/daily_aggregate_date_utils.py` (+66/-0) — new helper module (likely conversion shims).
- `litellm/proxy/db/db_spend_update_writer.py` (+9/-2), `litellm/proxy/guardrails/usage_endpoints.py` (+18/-11), `litellm/proxy/guardrails/usage_tracking.py` (+4/-2), `litellm/proxy/management_endpoints/common_daily_activity.py` (+15/-8), `litellm/proxy/management_endpoints/user_agent_analytics_endpoints.py` (+18/-7) — call-site updates to use the new `daily_aggregate_date_utils` shim.
- `litellm/types/proxy/management_endpoints/common_daily_activity.py` (+5/-3), `litellm/proxy/_types.py` (+4/-0) — type updates.
- `tests/test_litellm/proxy/db/test_daily_aggregate_date_utils.py` (+67/-0), `tests/test_litellm/proxy/db/test_db_spend_update_writer.py` (+7/-4) — new and updated tests.

## Specific observations

- `migration.sql:1-10` — the explanatory header comment ("text column has always been a date-of-day in YYYY-MM-DD format, so the cast is lossless") is exactly what an operator needs to read before running this. Good.
- `migration.sql` idempotency guard (`IF EXISTS ... data_type = 'text'`) is the correct pattern — re-running this migration after a successful pass will be a no-op. **However**, this only protects against re-running on already-migrated dbs; it does NOT protect against rows containing non-date strings. If any environment somehow has a malformed `date` value, `ALTER ... USING "date"::date` will hard-fail mid-table. Consider either a pre-flight `SELECT COUNT(*) FROM ... WHERE "date" !~ '^\d{4}-\d{2}-\d{2}$'` or wrap the cast in `TO_DATE("date", 'YYYY-MM-DD')` with explicit error handling.
- **Three identical `schema.prisma` files** (`schema.prisma`, `litellm/proxy/schema.prisma`, `litellm-proxy-extras/litellm_proxy_extras/schema.prisma`) all updated by hand. This is the root cause of recurring schema drift. Out of scope for this PR but worth filing a follow-up to dedupe to a single source of truth + generator.
- `schema.prisma`: `@client.Date` annotation maps to PG `DATE` (no time component). Application reads will now return Python `date` (not `datetime`), which is why the `daily_aggregate_date_utils.py` shim exists. Look carefully at every consumer that previously did `.strftime(...)` on the string — if any were assuming string semantics in JSON serialization, the response payloads will change shape (`"2026-05-03"` is the same in both cases, but `datetime.date.isoformat()` vs `str` behavior differs in pydantic v2 in some edge cases).
- `usage_endpoints.py` (+18/-11) and `common_daily_activity.py` (+15/-8) deserve careful read — these are the public-facing shapes. Any unintentional response-payload change is a contract break.
- The migration target list (8 tables) matches the schema.prisma diff. Good — no orphan tables left as TEXT.
- No down-migration. Postgres can't trivially revert `DATE → TEXT` without a USING cast either, so this is acceptable, but it should be explicit in the PR description: "this migration is forward-only".

## Verdict: `request-changes`

The substance of the migration is correct and the call-site updates look careful, but I'd hold for: (1) an explicit pre-flight check or `TO_DATE(...)` wrap to handle malformed data without a hard SQL failure, (2) PR description must spell out "forward-only" and recommend a pre-deploy `pg_dump` of the affected tables, (3) reviewers must walk every modified `usage_endpoints.py` / `common_daily_activity.py` line to confirm the JSON response shape is unchanged. The blast radius (8 tables, all spend tracking) makes "merge then fix" a poor strategy.
