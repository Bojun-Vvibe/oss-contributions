# BerriAI/litellm #26830 — fix(guardrails): show YAML-defined guardrails in Guardrail Monitor

- **URL:** https://github.com/BerriAI/litellm/pull/26830
- **Head SHA:** `d8c067413552a6ad57b48023dc9be3e69087b429`
- **Files:** `litellm/proxy/guardrails/{guardrail_registry,usage_endpoints}.py`, `tests/test_litellm/proxy/guardrails/test_usage_endpoints.py` (+411/-30, regression LIT-2529)
- **Verdict:** `merge-as-is`

## What changed

Guardrails defined in `config.yaml` were invisible to `/guardrails/usage/detail` (404) and only partially visible to `/guardrails/usage/overview` (no `type` / `description`) because both endpoints queried `prisma_client.db.litellm_guardrailstable.find_many()` and never consulted `IN_MEMORY_GUARDRAIL_HANDLER`. Three coordinated fixes:

1. **`initialize_guardrail` was dropping `guardrail_info`.** `guardrail_registry.py:487-491` adds `guardrail_info=guardrail.get("guardrail_info")` to the in-memory handler call, so YAML-defined `type` / `description` actually persist into `IN_MEMORY_GUARDRAILS`. This is the root-cause fix; without it the rest is a render of empty fields.
2. **Helper consolidation.** `usage_endpoints.py:140-145` introduces `_get_guardrail_field(g, field)` that handles both Prisma row (attr access) and dict / `Guardrail` TypedDict (key access). `:155-162` adds `_to_dict(value)` that coerces `LitellmParams` (Pydantic), dict, or `None` into a plain dict. Both replace ~12 inline `getattr-or-dict-get` ladders, removing a real bug class where `getattr(g, "X", None) or g.get("X")` mishandles falsy-but-present dict values.
3. **Endpoint behavior.** `/usage/overview` at `:281-296` now `union`s DB guardrails with `IN_MEMORY_GUARDRAIL_HANDLER.list_in_memory_guardrails()`, deduped by `guardrail_id` (DB wins, parity with `get_guardrail_info`). `/usage/detail` at `:359-365` falls back to the in-memory handler on DB miss, returning 404 only when both miss. `/usage/logs` at `:597-606` similarly falls back so the logical-name alias is included in the spend-log query.

## Why it's right

- The bug shape is canonical: two registries (DB rows vs in-memory) with overlapping IDs, but an endpoint reading from only one. The fix unions correctly with explicit dedupe semantics named in the comment at `:283-285` ("DB takes precedence").
- The `_get_guardrail_field` consolidation eliminates a subtle bug at the *previous* `_get_guardrail_attrs`: the old `getattr(g, "guardrail_id", None) or g.get("guardrail_id") if isinstance(g, dict) else None` short-circuits on a falsy but present `guardrail_id` (e.g. empty string from a misconfigured YAML). The new helper does pure presence-vs-absence and lets the caller decide the fallback (`name or gid or ""` at `:148`).
- Test coverage is dense and locks every behavioral edge:
  - `test_usage_detail_returns_yaml_guardrail` at `:301-313` — the headline regression case
  - `test_usage_detail_db_takes_precedence_over_yaml` at `:316-328` — locks the dedupe direction with `mock_in_memory.get_guardrail_by_id.assert_not_called()` (the right negative assertion)
  - `test_usage_overview_yaml_guardrail_metric_lookup_by_logical_name` at `:357-378` — exercises the metric-by-logical-name lookup AND asserts no orphan "Custom" row is double-emitted (`orphan_rows == []`)
  - `test_usage_overview_dedupe_when_guardrail_in_both_db_and_yaml` at `:381-401` — locks DB-precedence directly
  - `test_usage_logs_includes_logical_name_for_yaml_guardrail` at `:407-441` — asserts the `guardrail_id: {in: [...]}` filter contains *both* UUID and logical name (the actual contract the spend-log writer expects)
  - `test_usage_detail_with_real_in_memory_handler_preserves_guardrail_info` at `:447-485` — the integration test that exercises the *real* `IN_MEMORY_GUARDRAIL_HANDLER` to lock fix (1) end-to-end and would have caught the original `guardrail_info`-drop bug without any mocking
- The `mocker.patch(... "usage_endpoints.IN_MEMORY_GUARDRAIL_HANDLER", ..., create=True)` at `:282-286` is the right pattern — patches the *imported* symbol in the consumer module, not just the source module, so the tests reflect production import semantics.

## Nits

- Trivial: the `Guardrail` TypedDict import at `usage_endpoints.py:16` and the `LitellmParams` import next to it could be combined; not worth a re-roll.
- The integration test cleanup at `:484-485` does `IN_MEMORY_GUARDRAIL_HANDLER.IN_MEMORY_GUARDRAILS.pop(...)` directly — fine, but a `delete_in_memory_guardrail(guardrail_id)` helper on the handler would be a marginally better seam (and useful for the eventual hot-reload path).
- The `from fastapi import HTTPException` import inside the `/usage/detail` function at `:367` was already there; consider hoisting to module level since both endpoints raise it now.

## Risk

Very low. The endpoints are read-only, the union-and-dedupe logic is straightforward, and the test suite covers DB-only / YAML-only / both / neither / metric-lookup-by-logical-name / DB-precedence-on-collision exhaustively. The persistence fix at `guardrail_registry.py:491` is additive (passing through a previously-dropped field), no schema change.
