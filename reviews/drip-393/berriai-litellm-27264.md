# Review: BerriAI/litellm#27264

- **PR:** [perf(proxy): run daily activity aggregation off the event loop](https://github.com/BerriAI/litellm/pull/27264)
- **Head SHA:** `16920fba9cee5a30460d4a3cb6e674901c5f306f`
- **Merged:** 2026-05-06T03:19:29Z
- **Verdict:** `merge-after-nits`

## Summary

Two-part perf win for the daily-activity endpoint: (1) push every rollup level into Postgres via `GROUPING SETS` so all the per-date / per-(date,model) / per-(date,endpoint,api_key) buckets the response needs come back in one query pass tagged with a `GROUPING()` bitmask; (2) move the resulting Python dispatch loop into `asyncio.to_thread` so the event loop stays responsive during large aggregations. Reported result: median aggregation 10.43s → 8.96s, p99 liveness latency 9.0s → 1.45s.

## Specific notes

- `litellm/proxy/management_endpoints/common_daily_activity.py:547-595` — SQL switched from a single `GROUP BY (date, api_key, model, model_group, custom_llm_provider, mcp_namespaced_tool_name, endpoint)` to `GROUPING SETS (...)` with 13 explicit grouping sets plus the grand total `()`. The leaf grouping is intentionally omitted — comment at L546-552 explains this is because nothing in the response shape consumes it once the rollups are present. Good rationale, kept inline.
- `litellm/proxy/management_endpoints/common_daily_activity.py:657-664` — `GROUPING()` columns added to the SELECT, encoding which bits are rolled up. Standard Postgres semantics: rightmost arg = least significant bit; bit is 1 when the column is *not* part of the current grouping set's key.
- `litellm/proxy/management_endpoints/common_daily_activity.py:707-721` — `_GROUP_*` bitmask constants are spelled out individually with binary comments. This is exactly the right way to do it — readers can verify each constant against the column order in the SELECT without reverse-engineering bit math. Worth its weight.
- `litellm/proxy/management_endpoints/common_daily_activity.py:646-672` — `_aggregate_spend_records` now collects the `api_keys` set and fetches `api_key_metadata` *before* offloading to `asyncio.to_thread`, then hands the pre-fetched dict to the sync function. Right pattern: keep DB I/O on the event loop (Prisma client is async), keep CPU-bound row dispatch off it.
- `_aggregate_grouping_sets_records_sync` (introduced further down) dispatches each row directly into its bucket via the bitmask — no `update_metrics`-style nested summing, no per-row mutation of multiple buckets. This is what makes the perf win real, not just the threading.
- One concern: the grouping-set list is hand-maintained and the bitmask constants must stay in sync with the column order. Any future addition of a column to the SELECT will silently shift the bit positions. A unit test that asserts each `_GROUP_*` constant against `GROUPING()` for a known input would lock this down — worth a follow-up.

## Rationale

The change is well-motivated (real benchmark data attached), the SQL is correct (GROUPING SETS is the standard Postgres tool for this exact problem), and the threading model is right (DB I/O stays async, CPU dispatch goes to a thread). The bitmask approach is fragile to column-order changes but the inline binary-literal comments make it auditable; the missing piece is a test that pins the constants.

Nit: add a regression test that runs the SELECT against a fixture and asserts a few rows have the expected `group_level` bitmask. That's the only thing protecting the constants from silent drift. Not blocking the merge.
