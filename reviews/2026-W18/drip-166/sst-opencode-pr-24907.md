# sst/opencode #24907 — fix(session): exclude archived sessions from Session.list() query

- **PR:** https://github.com/sst/opencode/pull/24907
- **Head SHA:** `fd4bdc4bf7d27a9b8bd3c19184f4df2c8bd79f20`
- **Size:** +71 / −0 (1 production line, ~70 of test setup)

## Summary
`Session.list()` was returning archived sessions alongside active ones. The renderer treated those rows as live, leaking deleted-from-UI sessions back into the picker and (more subtly) silently consuming `limit` slots so the UI under-rendered active rows whenever archives were present. Fix adds an `archived?: boolean` opt-in input and, when not set, appends `isNull(SessionTable.time_archived)` to the WHERE conditions.

## Specific observations

1. **One-line production fix at `packages/opencode/src/session/session.ts:771-786`:**
   ```ts
   if (!input?.archived) {
     conditions.push(isNull(SessionTable.time_archived))
   }
   ```
   Inserted before the workspace/limit/search conditions, so it composes correctly with the rest of the query builder. Tri-state-vs-binary nit below.

2. **Three load-bearing tests at `test/server/session-list.test.ts:229-296`** cover exactly the right axes:
   - default-exclude (positive + negative assertion)
   - **limit-slot consumption** (`{limit: 2}` with one archived: asserts `sessions.length === 2` and that both active rows are returned — this is the real user-visible regression, not just "archived shows up")
   - opt-in re-inclusion via `{archived: true}`

   The limit-slot test is the one I'd specifically want to see for any "exclude X from list" fix, because that's where post-filter bugs hide. Filtering at the SQL level (which this does) is the correct shape — a JS-side post-filter would have failed the `limit: 2` assertion.

3. **The opt-in semantics are `archived?: boolean`** which collapses to two states (`!input?.archived` is true for both `false` and `undefined`). That means there's no way to ask for "only archived" — you can only ask for "active only" (default) or "active + archived" (`true`). For a session-restore UI that wants to show only archive, callers will have to client-side-filter the union. Worth a follow-up tri-state (`'active' | 'archived' | 'all'`) if any caller actually needs it.

4. **`time_archived` is the canonical column** — the rest of the file uses `time_archived` consistently (the test sets it via `db.update(SessionTable).set({ time_archived: Date.now() })` at line 238). No risk of "is_archived flag drift".

## Risks

- **Behavioral change for any external API caller** that was relying on `Session.list()` returning archived rows (e.g. a custom CLI script that wanted to see everything). Default flip is correct for the UI — but worth a one-line CHANGELOG note for SDK consumers.
- **`isNull` vs `is null` SQL semantics in SQLite** — `time_archived IS NULL` is what you want; `time_archived = NULL` would silently match nothing. Drizzle's `isNull` helper generates `IS NULL` correctly, so this is fine — flagging only because it's a common bug class for "filter out archived" queries written with raw SQL.
- **Index coverage:** if `SessionTable` has a partial/composite index on `project_id` + `time_archived`, the new predicate is free. If not, this adds a sequential filter to every list call. Worth a `EXPLAIN QUERY PLAN` check on a populous DB before tagging perf-blocking.

## Suggestions

- **(Recommended)** Promote `archived` to `'active' | 'archived' | 'all'` (default `'active'`) so callers can build an Archive view without two queries + client-side filtering. Adds one extra branch but fixes the asymmetric API.
- **(Recommended)** CHANGELOG / migration note: "`Session.list()` now excludes archived sessions by default. Pass `{archived: true}` to restore previous behavior."
- **(Optional)** Add `EXPLAIN QUERY PLAN` snapshot test or a perf microbench against a 1k-session DB to lock in that the new predicate uses an index, not a scan.

## Verdict: `merge-after-nits`

Correct fix at the right layer (SQL, not post-filter), with the exact test that catches the real user-visible bug class (`limit` slot consumption). The nits are API shape (two-state vs three-state opt-in) and a CHANGELOG entry — neither blocks merge.

## What I learned

When a "show only X" filter is added late, the load-bearing test isn't "X doesn't show up" — it's "X doesn't consume `limit`/`offset` slots." The first assertion catches the symptom; the second catches the bug class (post-filter vs SQL-filter). This PR's `limit: 2` test is the right shape, and any future "exclude Y" PR should adopt the same pattern.
