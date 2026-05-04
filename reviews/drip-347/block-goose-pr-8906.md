# block/goose#8906 — fix(provider_registry): case-insensitive model name lookup for context_limit

- **Head SHA**: `6efe4c2c073e477775251ddeb73a1766a631a391`
- **Verdict**: `merge-as-is`

## Summary

One-line behavior change in `crates/goose/src/providers/provider_registry.rs:63` — swaps `m.name == model.model_name` for `m.name.eq_ignore_ascii_case(&model.model_name)` so custom-provider configs with case-mismatched model IDs (e.g. `GLM-5.1` in JSON vs `glm-5.1` in `GOOSE_MODEL`) actually find their `context_limit` instead of silently falling back to the default 128k. Adds three unit tests. +64/-1. Fixes #8752.

## Findings

- `crates/goose/src/providers/provider_registry.rs:63`: the change is `m.name.eq_ignore_ascii_case(&model.model_name) && m.context_limit > 0`. ASCII-case-insensitivity is the correct choice here — model identifiers are conventionally ASCII and a full Unicode `to_lowercase` would be heavier and could cause locale-dependent surprises. Good call.
- `:283-293` (`make_test_entry`): clean test helper that stubs out the constructor / inventory closures with `unimplemented!()`. Acceptable since the tests only exercise `normalize_model_config`, which doesn't touch those fields.
- `:296-306` (`test_normalize_model_config_case_mismatch`): registry has `GLM-5.1` (32k), `GOOSE_MODEL` is `glm-5.1`, asserts the normalized config picks up `Some(32_000)`. This is the regression guard; without the fix it would assert `None`.
- `:309-318` (`test_normalize_model_config_exact_case_match`): defends against the trivial regression where someone "fixes" the comparison and accidentally breaks exact matches. Good belt-and-suspenders coverage.
- `:321-330` (`test_normalize_model_config_no_match_falls_back`): asserts unknown models keep `context_limit: None`. This guards against the failure mode where someone naively makes the lookup return *anything*.
- The `&& m.context_limit > 0` clause is preserved, so entries with `context_limit: 0` still skip — important for partial provider definitions.

## Recommendation

Surgical, well-tested, low-risk. Merge as-is.
