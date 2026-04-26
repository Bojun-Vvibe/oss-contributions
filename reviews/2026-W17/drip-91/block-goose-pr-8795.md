---
pr: 8795
repo: block/goose
sha: 7c3d5035a5f3646c12dba8eaf7d865fef4c998dc
verdict: merge-after-nits
date: 2026-04-27
---

# block/goose #8795 — fix(providers): honor dynamic_models: false in declarative provider configs

- **Head SHA**: `7c3d5035a5f3646c12dba8eaf7d865fef4c998dc`
- **URL**: https://github.com/block/goose/pull/8795
- **Size**: medium (332/12 across 3 files; ~half is tests)

## Summary
The `dynamic_models: Option<bool>` field on `DeclarativeProviderConfig` was previously parsed but unused — every provider always tried `/v1/models` first and only fell back to the static `models` list on a 404. This PR makes `dynamic_models: Some(false)` actually mean "skip the API call and return the static list", and rejects construction when `dynamic_models: false` is set but no static `models` are listed.

## Specific findings
- `crates/goose/src/config/declarative_providers.rs:75-83` — adds a doc comment on `dynamic_models` documenting the three cases (`Some(false)+models` returns static, `Some(true)/None` tries API). Comment matches the implementation. Good.
- `crates/goose/src/providers/anthropic.rs:97-114` and `crates/goose/src/providers/openai.rs:160-177` — same pattern duplicated across both providers: build `custom_models` from `config.models` first, then validate `dynamic_models == Some(false) && custom_models.is_none()` to fail-fast at construction. The error message ("`dynamic_models: false` but no static models listed; at least one entry in `models` is required") is precise enough that a user can act on it without reading source. **Nit**: the duplication is a code smell — this is identical logic in two files. A `validate_dynamic_models(...)` helper in `declarative_providers.rs` (or as a `DeclarativeProviderConfig::custom_models()` method that returns `Result<Option<Vec<String>>>`) would let both providers call one validator. As more provider engines get added (the `ProviderEngine` enum suggests Bedrock/etc are planned), this divergence will compound.
- `anthropic.rs:298-302` and `openai.rs:516-520` — the actual short-circuit in `fetch_supported_models()`: `if self.dynamic_models == Some(false) { return Ok(custom_models.clone()); }`. Correct. The `Ok(custom_models.clone())` is fine — `Vec<String>` clone here is a few bytes per cycle and `fetch_supported_models` is not hot-path.
- `anthropic.rs:438-455` (test `fetch_supported_models_dynamic_false_uses_static`) — uses a `MockServer` that responds 500 to *any* GET, so a regression that re-introduced the API call would observably fail (the test asserts the static list is returned anyway). Strong test design.
- `anthropic.rs:455-475` (test `fetch_supported_models_dynamic_true_hits_api`) — verifies the inverse: `dynamic_models: Some(true)` does call `/v1/models` and ignores the static list. The mock returns `["api-m1", "api-m2"]` and the assertion is that the static `["static-m1"]` is *not* the result. Catches the "did I read the flag with the wrong polarity" bug class.
- `anthropic.rs:475-491` (test `fetch_supported_models_none_falls_back_on_404`) — preserves the existing fallback behavior when `dynamic_models` is `None`. This is the migration-safety property for existing user configs.
- `anthropic.rs:494-507` (`from_custom_config_rejects_static_only_without_models`) — covers the validation error path. Asserts the error message contains the literal `"dynamic_models: false"` so the user-facing message is regression-tested too.
- The OpenAI side (`openai.rs:982+`) appears to have a parallel test block (`// ── dynamic_models behavior ──`) but I couldn't see its full contents in the visible diff; assuming it mirrors the Anthropic tests, that's the right shape. **Pin**: verify all four scenarios are covered on the OpenAI side too, not just the happy path.
- The `dynamic_models: Some(true)` branch hits the API even when `custom_models` is set. Worth a comment in the user-facing docs (not in this PR's scope) clarifying that a static `models` list under `dynamic_models: true` is effectively unused unless the API is unreachable.

## Risk
Low. Existing configs with no `dynamic_models` field set continue to take the `None` branch and behave exactly as before (try API → fall back to static on 404). The new behavior is opt-in via an explicit `dynamic_models: false`.

## Verdict
**merge-after-nits** — extract the duplicated `dynamic_models: false && empty models` validator into a single helper in `declarative_providers.rs`, and confirm the OpenAI test block mirrors all four Anthropic tests. Otherwise this is a clean fix to a config field that was silently no-op.
