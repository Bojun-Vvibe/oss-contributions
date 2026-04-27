# BerriAI/litellm#26648 — fix(guardrails): show YAML-defined guardrails in Guardrail Monitor

- **PR**: https://github.com/BerriAI/litellm/pull/26648
- **Author**: yuneng-berri (yuneng-jiang)
- **Head SHA**: `44923f260c06e845cc4b1579694cab4588d7725b`
- **Verdict**: `merge-after-nits`

## Context

The litellm proxy admits guardrail definitions from two parallel sources — a Prisma table `litellm_guardrailstable` (DB-defined, mutable via the admin UI) and the YAML config `litellm_settings.guardrails` (declarative, loaded into the in-memory `IN_MEMORY_GUARDRAIL_HANDLER` registry at boot). The Guardrail Monitor admin endpoints at `litellm/proxy/guardrails/usage_endpoints.py` (`/guardrails/usage`, `/guardrails/usage/<id>`, `/guardrails/usage/logs`) only ever queried the Prisma table, so users with YAML-defined guardrails saw an empty Monitor UI even though the guardrails were running and logging metrics. This PR unions the two sources so the Monitor sees both.

## What it changes

Three coordinated additions plus a consolidation refactor of repetitive field-extraction code. New imports at `:14-15` pull in `IN_MEMORY_GUARDRAIL_HANDLER` and `LitellmParams`. Two new helpers at `:151-164`: `_get_guardrail_field(g, field)` reads a field via `g.get(field)` for dicts and `getattr(g, field, None)` for Prisma rows, abstracting over the polymorphism; `_to_dict(value)` coerces `LitellmParams` (via `model_dump(exclude_none=True)`), passes through `dict`, and returns `{}` for everything else. The `_guardrail_overview_rows` function at `:185-189` collapses six lines of `(g.litellm_params or {}) if isinstance(g.litellm_params, dict) else {}` into two `_to_dict(_get_guardrail_field(g, ...))` calls. The behavior change is at `:286-301` where `guardrails_usage_overview` now does `db_guardrails = await prisma_client.db.litellm_guardrailstable.find_many()`, computes `seen_ids = {getattr(g, "guardrail_id", None) for g in db_guardrails if ...}`, then appends `IN_MEMORY_GUARDRAIL_HANDLER.list_in_memory_guardrails()` filtered to entries whose `guardrail_id` is not in `seen_ids`. The detail endpoint at `:338-348` falls back from the Prisma `find_unique` to `IN_MEMORY_GUARDRAIL_HANDLER.get_guardrail_by_id(guardrail_id=...)` and only `404`s if both miss. The shape consolidation continues at `:411-415` (detail-endpoint field reads) — three repeated `getattr(...) or guardrail.get(...)` chains become three `_get_guardrail_field` calls.

## Design analysis

The dedup-by-`guardrail_id` strategy at `:289-301` is the right precedence: DB wins on collision (because YAML-loaded guardrails are filtered out *if their ID already appears in the DB set*), which matches the rest of the proxy's "DB is the live source of truth, YAML is the bootstrap default" convention. The `_to_dict` helper correctly handles the `LitellmParams` Pydantic-model case via `model_dump(exclude_none=True)` — this is the load-bearing detail because `IN_MEMORY_GUARDRAIL_HANDLER.list_in_memory_guardrails()` returns dicts whose `litellm_params` field is a `LitellmParams` instance (not a plain dict, unlike the Prisma row's `Json` column), and the previous `isinstance(g.litellm_params, dict)` check would have silently coerced it to `{}` and lost the `guardrail` provider key needed for the overview row. The detail-endpoint fallback at `:340-343` correctly uses `if guardrail is None` (not the prior `if not guardrail`, which would also fire for any falsy non-None value) — this is a small but real correctness improvement because the Prisma client's missing-row return is `None`, and a row with all-default-falsy fields would have wrongly tripped the old branch.

## Concerns / nits

1. **`getattr(g, "guardrail_id", None) for g in db_guardrails` runs twice on the set comprehension at `:291-294`** — once in the `for` clause and once in the `if`. Trivial cost, but the more idiomatic shape is `seen_ids = {gid for g in db_guardrails if (gid := getattr(g, "guardrail_id", None))}` (Python 3.8+ walrus). Not blocking.

2. **`_to_dict` doesn't handle the `BaseModel` general case** — only `LitellmParams` specifically. If a future guardrail subclass returns a different Pydantic model from its `litellm_params` accessor, this will silently coerce to `{}`. Worth either broadening to `isinstance(value, BaseModel)` (and importing `pydantic.BaseModel`) or naming the contract in the docstring ("only `LitellmParams` instances are supported; other Pydantic models are coerced to `{}` and will lose data").

3. **`logical_id = _get_guardrail_field(guardrail, "guardrail_name")` at `:413`** is functionally equivalent to the prior expanded form, but the prior code had an `or guardrail_id` fallback hidden in the next line (`metric_ids = [i for i in (logical_id, guardrail_id) if i]`); the new code preserves this via the same `metric_ids` list comprehension at `:415`, so behavior is unchanged. Worth a one-line comment naming the contract ("metrics are keyed by logical name *or* falling back to UUID, both candidates feed `metric_ids`") to prevent a future refactor from collapsing the list comp and silently dropping the fallback.

4. **No new test coverage** for the YAML-guardrail visibility path. The change is small enough that this is reviewable-by-inspection, but a single integration test that loads a YAML config with one guardrail, hits `/guardrails/usage`, and asserts the YAML guardrail appears would close the regression-detection gap and document the contract. Worth asking for in the PR but not blocking if the maintainer is comfortable with the risk profile.

5. **`IN_MEMORY_GUARDRAIL_HANDLER.list_in_memory_guardrails()` return shape contract** — the iteration at `:298-301` assumes the return is a list of dicts (uses `.get("guardrail_id")`). If a future refactor of the in-memory handler changes the return to a list of `LitellmParams` (or something else), this loop silently filters everything out. A `# contract: returns List[Dict[str, Any]] keyed by guardrail_id` comment would document the load-bearing assumption.

## Risks

Medium-low. The behavior change is strict-monotone (the API now returns *more* guardrails than before, never fewer), so no existing client breaks. The `_get_guardrail_field` / `_to_dict` consolidation is an exact behavioral substitute for the prior inlined form (verified by inspection) — the new helpers handle dict, Prisma row, `LitellmParams`, and the no-op fallback identically to what they replace. The detail-endpoint fallback at `:340-343` does add a new `IN_MEMORY_GUARDRAIL_HANDLER.get_guardrail_by_id` call site, which means if the in-memory handler raises (rather than returning `None`) on a missing key, the endpoint will 500 instead of 404 — worth verifying that `get_guardrail_by_id` returns `None` on miss (the call shape suggests it does, but the diff doesn't pin it).

## Suggestions

Land after addressing nits 2 + 4 (broaden `_to_dict` or document the narrow contract; add a single integration test for the YAML-guardrail visibility path). The other three nits are nice-to-have and can land in a follow-up.

## What I learned

The "DB and YAML are both sources, dedup by ID with DB winning" pattern is recurring across the litellm proxy surface (model registry does the same). The `_get_guardrail_field` + `_to_dict` helper pair is a clean extraction of the polymorphic-row-shape problem and should be reusable in other admin endpoints that mix Prisma rows with in-memory registry entries. Worth checking whether `litellm/proxy/management_endpoints/` has the same shape repeated and could share these helpers.
