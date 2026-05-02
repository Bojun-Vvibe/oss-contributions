# Review: openai/codex #20765

- **PR:** openai/codex#20765 — `[codex] expose app-server config flag for local image resize preferences`
- **Head SHA:** `2394ba310d00c65fb9d6519008d2dcca9cd83610`
- **Files:** 9 (+188/-71)
- **Verdict:** `merge-after-nits`

## Observations

1. **`LocalImageResizePolicy` enum + `LocalImageConfig` wrapper is well-scoped** — `config_toml.rs` adds the enum with `#[serde(rename_all = "snake_case")]` and `#[default] ResizeToFit`; `core/src/config/mod.rs` introduces a `LocalImageConfig` struct that pairs the policy with `max_dimension` (default `MAX_DIMENSION` from `codex_utils_image`). The `prompt_image_mode()` adapter cleanly translates to the existing `PromptImageMode` enum so callers in `session/turn.rs` and `tools/handlers/view_image.rs` don't need to know about toml shape.

2. **Validation is correct but incomplete** — `resolve_local_image_config` rejects `max_dimension == 0` with `InvalidInput`. Good. But there's no upper bound, so a user setting `local_image_max_dimension = 4294967295` will allocate accordingly inside the resizer. Consider clamping to a sane max (e.g. 8192 or whatever the model's actual limit is). The `#[schemars(range(min = 1))]` annotation on `config_toml.rs` only constrains the JSON schema, not deserialization.

3. **App-server README documentation (`README.md:619`)** correctly notes both keys and the `2048` default — but it says "thread config keys" while the implementation reads them from `ConfigToml` (base config), not per-thread overrides. Either the doc is wrong or the per-thread routing is missing. Worth verifying whether `ConfigOverrides` carries these through `ThreadConfigLoader`.

4. **Test coverage adds the new field to four `_precedence_fixture_*` tests** but does not add a positive test that exercises a non-default `local_image_resize_policy = "original"` flowing through to `prompt_image_mode()`. A small unit test on `LocalImageConfig::prompt_image_mode` matrix (both variants) would lock in the mapping.

## Recommendation

Land after the upper-bound clamp + doc/per-thread clarification; the design is good.
