# BerriAI/litellm PR #27070 — fix(ui): omit unchanged allowed_routes on key update

- **Repo:** BerriAI/litellm
- **PR:** #27070
- **Head SHA:** `483c20ee865b7a1ced6a318b1df502f27f9d64d7`
- **Author:** fengfeng-zi
- **Title:** fix(ui): omit unchanged allowed_routes on key update
- **Diff size:** +94 / -0 across 2 files
- **Drip:** drip-292

## Files changed

- `ui/litellm-dashboard/src/components/templates/key_info_view.tsx` (+34/-0) — adds `normalizeStringList`, `areStringListsEqual`, and a guard inside `onSubmit` that strips `allowed_routes` from the update payload when normalized old/new lists are equal.
- `ui/litellm-dashboard/src/components/templates/key_info_view.test.tsx` (+60/-0) — three regression tests covering: unchanged value, empty-on-empty, and clear-from-non-empty.

## Specific observations

- `key_info_view.tsx:79-94` (`normalizeStringList`) — trims and filters empty entries from arrays *and* CSV strings. Correct for the proxy admin form where the input control may emit either shape. One nit: the CSV branch silently swallows whitespace-only entries; that is the intended behavior here but worth a one-line comment so a future maintainer doesn't "fix" it.
- `key_info_view.tsx:96-104` (`areStringListsEqual`) — order-sensitive comparison. The route override is conceptually a *set*, not a *list*, so two equivalent overrides in a different order (`["x","y"]` vs `["y","x"]`) will still trigger an update. Either sort both sides before compare, or document explicitly that order is significant.
- `key_info_view.tsx:113-118` — the strip happens *after* the `formValues.allowed_routes` mutation but *before* `max_budget` mapping. Placement is fine; the inline comment ("non-admin editors don't trip the backend permission check on a no-op save") accurately captures the intent.
- `key_info_view.test.tsx:10-28` — first test ("drop allowed_routes when submitted matches existing") asserts `expect.not.objectContaining({ allowed_routes: expect.anything() })`. Good; this is the strict assertion (key absent, not just falsy).
- `key_info_view.test.tsx:50-68` — third test correctly verifies that a *real* clear (non-empty → empty) still propagates `allowed_routes: []`. This is the critical inverse case and it's covered.
- Missing test: there's no case for `allowed_routes` going from `["a"]` to `["a","b"]` (a real edit). The existing test suite presumably covers basic edits but adding one explicit case would round out the matrix.
- The change is UI-only and additive; no new deps, no API surface change.

## Verdict: `merge-after-nits`

Solid fix for #27005 with the right test shape. Two small asks: (1) decide whether `allowed_routes` is order-sensitive and either sort or document, and (2) add one positive-edit regression test alongside the three no-op cases. The set-vs-list ambiguity is the only thing that could cause a follow-up bug.
