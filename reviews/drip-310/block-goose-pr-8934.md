# block/goose #8934 — feat(recipe): support structured parameters (object/array) in recipe templates

- PR: https://github.com/block/goose/pull/8934
- Author: jordigilh
- Head SHA: `826a5cf11fcb10510458352c35402df99e4877d6`
- Updated: ~2026-05-01
- Discussion: https://github.com/block/goose/discussions/8917

## Summary
Adds `Object` and `Array` variants to `RecipeParameterInputType` and wires structured parameter support end-to-end through `build_recipe_from_template`. Callers continue to pass `Vec<(String, String)>`; the build pipeline detects object/array params, parses their string values as `serde_json::Value`, and routes through a new `render_recipe_content_with_structured_params` MiniJinja path. Existing string-only callers keep the existing fast path (no JSON parsing overhead).

## Observations
- `crates/goose/src/recipe/build_recipe/mod.rs:35-50`: `has_structured_params` is computed by scanning `recipe_parameters` once; if no object/array param exists, the existing `render_recipe_content_with_params` path is taken untouched. Zero-overhead-when-unused is the correct migration shape for a feature that touches a hot pipeline.
- `crates/goose/src/recipe/build_recipe/mod.rs:57-91`: `to_structured_params` enforces type-correctness on parsed JSON — an `input_type: object` parameter whose JSON parses to an array (or vice versa) is rejected with a clear error including `json_type_name(&parsed)` ("an array", "a number", etc.). That's a *much* better UX than letting MiniJinja blow up with an opaque attribute-access error several layers down.
- `crates/goose/src/recipe/build_recipe/mod.rs:67-77`: the JSON parse error message includes both the parameter key and the upstream `serde_json::Error`. Good.
- The string-keyed `structured_keys: HashMap<&str, &RecipeParameterInputType>` borrows from `recipe_parameters` for the duration of the closure — fine, lifetimes look correct from the snippet.
- `crates/goose/src/recipe/build_recipe/mod.rs:101-110`: `json_type_name` covers all six `serde_json::Value` variants exhaustively. Consider using a `match` arm-list with `=> "..."` rather than the if/else chain (already done — good).
- Fallback for non-structured keys is `Value::String(v.clone())`. That means a `string`-typed param shows up in the structured map as a JSON string, which MiniJinja will treat as an attribute-less scalar — exactly matching the existing `render_recipe_content_with_params` behavior. Good — no semantic drift.
- `crates/goose/src/recipe/build_recipe/tests.rs:667-790`: four integration tests cover the happy paths (object dot-notation, array iteration, mixed string+object) and one error path (invalid JSON). Confirm the *type-mismatch* path (`input_type: object` + JSON array value) is also tested — the PR description claims 4 integration tests + 10 unit tests; from the diff snippet I see object/array/mixed/invalid-JSON but not the type-mismatch case. If it's not covered in `template_recipe.rs` either, please add one — the runtime guard exists but is uncovered.
- API surface preservation: `build_recipe_from_template` signature unchanged, all existing callers (`goose-cli`, `goose-server`, `summon`, `execute_commands`) get the feature for free. Excellent backward-compat story.
- Security note in PR body: same MiniJinja `Environment` (Strict undefined, recipe-dir-scoped loader) for the new render path. No new attack surface — the fix is purely a richer value type passed to the same engine.
- Documentation: `documentation/docs/guides/recipes/recipe-reference.md` is updated per the file changes table. Verify the doc actually shows a structured-param example with the JSON-string-passing convention (since callers still pass strings) — that's the most likely point of confusion for users.
- Scope/limitations called out honestly in the PR body: Desktop/CLI UX falls back to text input for unknown types. That's the right initial cut; JSON-aware widgets can be a follow-up.
- `cargo fmt`, `cargo clippy`, `cargo check -p goose --lib` all reportedly clean.

## Verdict
`merge-after-nits`
