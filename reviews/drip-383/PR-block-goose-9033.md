# block/goose#9033 — fix: case-insensitive model name lookup for context_limit resolution

- **Head SHA**: `ef6897674ae25d018f2afaba3e6e44c7f2fc57a6`
- **Author**: treebird7 (Treebird)
- **Stats**: +181 / -5, 2 files (relates to #8752)

## Summary

Custom providers store model names with original casing (`GLM-4.5`) while `GOOSE_MODEL` in `config.yaml` is lowercased (`glm-4.5`). The mismatch caused `context_limit` lookups to silently fall back to a 128k default. PR introduces a two-tier lookup: exact match first (preserves O(1) HashMap behavior and disambiguates case-distinct entries like `Kimi-K2-Instruct` 256k vs `kimi-k2-instruct` 128k), then a case-insensitive scan with deterministic tie-break via `min_by_key` lexicographic ordering.

## Specific citations

- `crates/goose/src/model.rs:31-43`: new `find_in_models(models, model_name)` helper. Exact `m.name == model_name` first, else `to_lowercase()` comparison. Correctly preserves case-distinct registrations.
- `model.rs:46-48`: `find_predefined_model` now delegates to the helper — keeps the public API stable, single source of truth for the lookup logic.
- `crates/goose/src/providers/canonical/registry.rs:78-83`: `register()` is unchanged in behavior (stores with original casing) — comment at `:79-80` documents the invariant.
- `registry.rs:85-105` is the load-bearing fix: `get()` does exact lookup first, then case-insensitive scan with `min_by_key(|((p, m), _)| (p.as_str(), m.as_str()))` for deterministic tie-break. The comment at `:91-95` correctly identifies the HashMap-iteration-order non-determinism that `.find()` would have introduced.
- `registry.rs:106-111`: `get_all_models_for_provider` also lowercases for filtering — extends the case-insensitivity guarantee uniformly.
- Tests:
  - `model.rs:594-637`: `test_find_predefined_model_case_insensitive` (3 casings + miss) and `test_find_in_models_exact_match_wins` (case-distinct dataset returns correct context_limit per query).
  - `registry.rs:155-225`: 3 tests — `test_get_is_case_insensitive`, `test_register_preserves_case_distinct_entries` (asserts `count() == 2` and per-key context limits), `test_ambiguous_case_fallback_is_deterministic` (calls `get()` twice with mixed-case query, asserts equality).

## Verdict

**merge-as-is**

## Rationale

Excellent fix. The two-tier strategy is the right call — naïve "lowercase everything on insert" would have collapsed the legitimate case-distinct entries (`Kimi-K2-Instruct` vs `kimi-k2-instruct`) silently. The deterministic-tie-break via `min_by_key` is a thoughtful detail that prevents flaky tests / non-reproducible bugs in the ambiguous-fallback case. Test coverage is thorough and asserts both correctness *and* invariants (count, determinism, case-distinct preservation). Comments at every non-obvious site (`:79-80`, `:91-95`, `:594-596`) name the rationale rather than just describing the code. The only theoretical concern is a slight perf cost on the case-insensitive fallback (O(N) scan with `to_lowercase()` allocations per key), but this only fires on cache miss after the O(1) exact-match check, and the registry is small (model count, not request count). Ship it.

