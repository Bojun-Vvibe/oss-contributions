# block/goose#8906 — fix(provider_registry): case-insensitive model name lookup for context_limit

- **PR**: https://github.com/block/goose/pull/8906
- **Head SHA**: `6efe4c2c073e477775251ddeb73a1766a631a391`
- **Size**: +64 / −1, 1 file
- **Files**: `crates/goose/src/providers/provider_registry.rs`
- **Fixes**: #8752

## Context

Custom-provider model registry entries with case-mismatched model names (e.g., `GLM-5.1` in the provider's JSON config but `glm-5.1` in `GOOSE_MODEL`) silently failed the strict-equality lookup in `normalize_model_config`, falling back to the default 128k `context_limit`. This caused users with mixed-case model names (common in Chinese-provider naming conventions like ZhipuAI's GLM family) to get the wrong context window, with no error message indicating the mismatch.

The fix is a one-character change to use `eq_ignore_ascii_case` instead of `==` in the lookup, plus 64 lines of unit tests covering case-mismatch, exact-match, and unknown-model paths.

## Why this exists / problem solved

Model name registries from upstream provider docs are inconsistent in case: OpenAI is canonically lowercase (`gpt-4o`), Anthropic is title-case-ish (`claude-3-5-sonnet`), and ZhipuAI/Moonshot/Qwen often ship with mixed case (`GLM-5.1`, `Qwen-2.5-Max`). When a user pulls a model name from a provider's docs and pastes it into `GOOSE_MODEL`, the case may not match what the provider entry's `known_models` list has. The strict `==` comparison at `provider_registry.rs:63` was the source of the silent fallback.

This is the smallest correct fix: change the comparator only. Don't normalize the stored model names (preserves what gets sent to the provider API), don't case-fold the user input (preserves the user's typed string for any other downstream uses). Only the *lookup* loosens.

## Design analysis

### The 1-line fix

At `crates/goose/src/providers/provider_registry.rs:63`:

```rust
- .find(|m| m.name == model.model_name && m.context_limit > 0)
+ .find(|m| m.name.eq_ignore_ascii_case(&model.model_name) && m.context_limit > 0)
```

`eq_ignore_ascii_case` is the right choice over `to_lowercase().eq(...)` because:

1. **Zero allocation** — `eq_ignore_ascii_case` walks the bytes pairwise; `to_lowercase()` allocates a new `String`.
2. **No locale dependence** — model names are ASCII by convention (the regex enforced by major providers excludes non-ASCII), so ASCII-only case folding is the right semantics. Using `to_lowercase()` would do Unicode-correct folding which is *more* permissive in pathological ways (Turkish dotted-i, German sharp-s, etc.) and slower.
3. **Symmetric** — `a.eq_ignore_ascii_case(b)` returns the same as `b.eq_ignore_ascii_case(a)`, so it doesn't matter which operand is the registry entry vs the user input.

The `m.context_limit > 0` part of the predicate is unchanged — that's the existing skip-zero-limit guard which keeps the lookup from matching placeholder entries with `context_limit: 0`.

### The 3 unit tests

`tests` module at `:283-345`:

1. **`test_normalize_model_config_case_mismatch`** at `:312-319` — registry has `"GLM-5.1"`, user requests `"glm-5.1"`, asserts `context_limit == Some(32_000)` (the registered value, found despite case mismatch). The exact bug scenario.
2. **`test_normalize_model_config_exact_case_match`** at `:322-329` — registry has `"gpt-4o"`, user requests `"gpt-4o"`, asserts `context_limit == Some(128_000)`. Regression-fence — confirms the existing happy path still works.
3. **`test_normalize_model_config_no_match_falls_back`** at `:332-340` — registry has `"GLM-5.1"`, user requests `"unknown-model"`, asserts `context_limit` stays `None`. Regression-fence for the "no entry matches → fall through to whatever the default is" path.

Three tests, three orthogonal cases. Each test is ~7 lines plus the shared `make_test_entry` helper at `:286-308`. The helper builds a synthetic `ProviderEntry` with sensible default values for every required field (constructor unimplemented, inventory_identity returning a default, etc.) — that's the right test-fixture shape for a registry-lookup test where 90% of the entry's fields are irrelevant.

The `ModelConfig::new_or_fail("...")` constructor is used at `:315`, `:325`, `:336`. The `_or_fail` suffix suggests it panics on invalid input — fine for tests where the input strings are static valid identifiers.

### What's not changed

The PR explicitly avoids:

1. **Mutating `model.model_name` to the registry's casing**. So if the user has `GOOSE_MODEL=glm-5.1`, that exact lowercase string is what gets sent to the provider API in the request. Critical for providers that *are* case-sensitive on the wire (the user's working casing is preserved); critical for logs/telemetry that key on the user's typed string.
2. **Normalizing the registry-side `m.name` casing**. Whatever the provider author wrote in their JSON config is preserved — so a registry entry of `"GLM-5.1"` displays in any error/debug output as `"GLM-5.1"`, matching what the user sees in provider docs.
3. **Touching native providers** (Anthropic, OpenAI). The PR body claims "no regression for native providers where case is already lowercase" — true by inspection. The behavior change only manifests when the registry-side and user-side strings differ by case; for already-matching cases, the predicate result is identical.

This is the right scope. A more aggressive fix would have normalized both sides at insertion/lookup time, which is a bigger semantic shift and risks breaking case-sensitive downstream consumers.

## Risks

- **Two registered models that differ only in case.** If a registry contains both `"GLM-5.1"` and `"glm-5.1"` (extremely unlikely but possible if two custom-provider configs are concatenated), `find()` returns the *first* match in iteration order — which is now non-deterministic relative to user intent (the user typing `glm-5.1` previously got the exact match; now they get whichever appears first in the registry). The `m.context_limit > 0` guard mitigates if one is a placeholder, but worth a test that pins the iteration-order behavior or, defensively, prefers exact-match-then-case-insensitive-match. (Probably not worth the complexity for a non-existent real-world conflict.)
- **Non-ASCII model names.** `eq_ignore_ascii_case` only folds A-Z ↔ a-z. A model named `"模型-1"` in the registry vs `"模型-1"` in user input — they match exactly already, no folding needed. A registry name of `"Model-Á"` vs user `"model-á"` — the ASCII-only fold leaves the non-ASCII letters' case mismatched, so the lookup still fails. Edge case; non-blocking. The existing strict-eq path also failed here.
- **No deprecation path for case-sensitive consumers.** If anyone *depended* on the strict case-sensitivity (e.g., to deliberately register `"foo"` and `"FOO"` as different models with different limits), this change silently breaks them. Plausibility is essentially zero, but a CHANGELOG entry noting "model name lookup is now case-insensitive in `normalize_model_config`" closes the surprise gap.
- **The `find` call now does case-folding on every iteration.** For a provider with 50 models, that's 50 byte-by-byte case-folded comparisons. Negligible; this is a startup-or-config-reload code path, not a request-time hot path.

## Suggestions

1. **Add a "duplicate registry entries differing only in case" test** — register `[ModelInfo::new("GLM-5.1", 32000), ModelInfo::new("glm-5.1", 64000)]`, request `"glm-5.1"`, assert *which* `context_limit` you get. If the answer is "first one wins (32000)," document it; if you want exact-match-preferred semantics, restructure the predicate as a two-pass scan (`find(|m| m.name == ...) ?? find(|m| m.name.eq_ignore_ascii_case(...))`). Probably overkill — pin the current behavior with a test and move on.
2. **CHANGELOG one-liner** — "`normalize_model_config` now performs case-insensitive lookup against `known_models` (#8752)." Documents the change for anyone whose dashboards key on registry behavior.
3. **Consider applying the same loosening at any sibling lookup sites** in `provider_registry.rs` or related modules. The PR fixes one specific `find` call; if there's a parallel lookup elsewhere (e.g., for default-model resolution, model-existence-check) that also uses strict `==`, it should get the same treatment in the same PR for consistency. Quick scan: `rg 'm\.name\s*==' crates/goose/src/providers/` or similar. If a sibling exists, fix it; if not, this is fully scoped.
4. **Doc-comment on `normalize_model_config`** — three lines saying "performs case-insensitive lookup of `model.model_name` against the registry's `known_models`; preserves the user's casing in the returned `ModelConfig`." Pins the contract so future contributors don't accidentally tighten the comparator back to `==`.
5. **(Optional) extend to a simple punctuation-tolerance** — some providers' model names use both `-` and `_` interchangeably (`gpt-4o` vs `gpt_4o`). Probably out of scope; users rarely hit this. Mention only if maintainer is interested in a broader normalization pass.

## Verdict

**merge-as-is** — one-character fix to the right comparator, three orthogonal unit tests covering the bug, the regression-fence happy path, and the no-match fallback. The shared `make_test_entry` helper at `:286-308` is a clean fixture pattern that future tests can reuse. The only "asks" are CHANGELOG-and-doc improvements, none of which gate the correctness of the fix. Ship it.

## What I learned

`eq_ignore_ascii_case` vs `.to_lowercase().eq(...)` is the small choice that separates production-quality Rust string code from "works in the happy path" Rust string code. The first is byte-level, allocation-free, and locale-independent — it pairs naturally with the kind of identifier comparisons (model names, tool names, header names) where the inputs are constrained to ASCII by convention. The second is correct in intent but wrong in performance and surprising in pathological inputs (Turkish-locale dotted-i is the canonical example of how `to_lowercase` differs from ASCII fold). For *any* identifier-style string comparison in Rust, the right reflex is `eq_ignore_ascii_case`; reach for `to_lowercase` only when you're working with user-facing text where you've consciously decided locale-correct folding is the right semantics. This PR makes the right choice without explanation, which is the typical signal of an experienced Rust contributor — but for an OSS repo where the PR will educate future contributors, a one-line `// ASCII fold: model names are ASCII by convention; locale-aware folding would be wrong here.` comment would extract maximum educational value from a one-character diff.
