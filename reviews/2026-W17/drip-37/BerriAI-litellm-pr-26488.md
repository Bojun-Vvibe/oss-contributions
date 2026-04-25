# BerriAI/litellm #26488 — UI Spend Logs sortable Model and TTFT columns

- **Repo**: BerriAI/litellm
- **PR**: [#26488](https://github.com/BerriAI/litellm/pull/26488)
- **Head SHA**: `2047446546251abe4caed657e5e3ffc9da414c96`
- **Author**: yuneng-berri
- **State**: OPEN (+274 / -10)
- **Verdict**: `merge-after-nits`

## Context

`/spend/logs/ui` already supports `sort_by` for Time, Cost,
Tokens, and Duration. Model and TTFT (time-to-first-token) headers
in the UI were unsortable. This PR extends the whitelist + adds a
synthetic `ttft_ms` ORDER BY expression for non-streaming-aware
sorting.

## Design

Backend in
`litellm/proxy/spend_tracking/spend_management_endpoints.py`:

1. Whitelist extension (lines 1798-1802 of the file, lines 12-14
   of the diff): `valid_sort_fields` set adds `"model"` and
   `"ttft_ms"`. Whitelist-then-format is the right shape — the
   value is interpolated directly into the SQL string later, so a
   raw allow-list plus a typed branch is the only way this is
   safe.
2. ORDER BY builder (lines 2047-2071 of the file, lines 28-49 of
   the diff): the previous one-liner that just quoted
   `startTime`/`endTime` is replaced with a 3-branch select:
   - `ttft_ms` → `CASE WHEN "completionStartTime" IS NULL OR "completionStartTime" = "endTime" THEN NULL ELSE (EXTRACT(EPOCH FROM ("completionStartTime" - "startTime")) * 1000) END`
     plus `NULLS LAST`. Non-streaming rows always sort to the
     bottom regardless of direction — matches the existing `-`
     UI display for those rows.
   - `startTime` / `endTime` → quoted column name (existing path).
   - Otherwise → bare column name (also existing path; safe
     because `valid_sort_fields` is the only source).
3. Frontend: adds `model` and `ttft_ms` to `LOGS_SORT_FIELD_MAP`
   and wraps the Model and TTFT (s) headers with the existing
   `SortableHeader` component. (Visible from PR description, not
   in the file diff snippet I have.)

Test coverage in
`tests/test_litellm/proxy/spend_tracking/test_spend_management_endpoints.py`:

- `test_ui_view_spend_logs_sort_by_model` (parametrized
  asc/desc): asserts the SQL contains `model` and *does not*
  contain `NULLS LAST` — that's a defensive assertion against
  accidentally widening the NULLS-LAST clause to all sort
  columns. Good narrow guard.
- `test_ui_view_spend_logs_sort_by_ttft_ms` (not shown in my
  diff window but mentioned in PR description): covers
  streaming + non-streaming rows in both directions, asserting
  NULL rows always sort last.

## Risks

1. **SQL injection surface.** `_order_expr` for the bare-column
   branch interpolates `order_column` directly. This is gated by
   the `valid_sort_fields` set check earlier — but the gate is in
   a different code section, several hundred lines away. A
   defensive comment near the `f"""...ORDER BY {_order_expr}..."""`
   pointing at the gate would help future readers. Not a real
   vulnerability today.
2. **`NULLS LAST` semantics with DESC.** PostgreSQL default is
   `NULLS FIRST` for `DESC` and `NULLS LAST` for `ASC` — by
   forcing `NULLS LAST` always, the desc-direction behavior on
   non-streaming rows changes from "show first" to "show last".
   This is actually the *desired* behavior per the PR rationale
   ("non-streaming rows always sort to the bottom regardless of
   direction"), and the test pins it. Just worth being explicit
   in the docstring.
3. **Index coverage.** Sorting by `model` on a large
   `LiteLLM_SpendLogs` table without an index on `model` is a
   sequential scan + sort. PR description says "no new indexes" —
   probably fine for typical UI page sizes (LIMIT/OFFSET is
   already there) but worth flagging in the release note for
   high-volume operators.

## Suggestions

- Add a `# safe: order_column is constrained by valid_sort_fields`
  inline comment next to the `_order_expr = order_column` branch.
- Consider whether `ttft_ms` should also accept the
  pre-computed `request_duration_ms`-style coalesced column if a
  future migration adds a stored TTFT column — the synthetic
  `EXTRACT(EPOCH FROM ...)` math will need re-pointing then. A
  brief comment on the CASE expression saying "synthetic until
  TTFT is materialized" would help.
- The `ttft_ms` "non-streaming row detection" uses
  `completionStartTime IS NULL OR completionStartTime = endTime`.
  The `=` branch handles a case the schema apparently
  back-fills with `endTime` for non-streaming completions; worth
  a comment confirming that's the intended invariant.

## What I learned

The "test asserts SQL contains string X *and not* string Y"
pattern (`assert "NULLS LAST" not in sql_query` in the model
test) is a nice cheap guard against a refactor accidentally
widening the special case. It's faster and more targeted than
asserting on row order, and it catches the most common
"someone moved the clause to the wrong branch" mistake.
