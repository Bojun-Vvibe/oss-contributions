# openai/codex #20765 — [codex] expose app-server config flag for local image resize preferences

- URL: https://github.com/openai/codex/pull/20765
- Head SHA: `2394ba310d00c65fb9d6519008d2dcca9cd83610`
- Author: @vivi
- Stats: +188 / -71 across 9 files

## Summary

Promotes the existing internal image resize defaults (`MAX_DIMENSION`,
`PromptImageMode`) into a first-class `local_image` config block plumbed
through the app-server so callers can opt for `resize_to_fit` (default,
2048px) or `original` per thread.

## Specific feedback

- `codex-rs/config/src/config_toml.rs:90-97` — new
  `LocalImageResizePolicy` enum is `#[serde(rename_all = "snake_case")]`,
  matches the documented `"resize_to_fit" | "original"` strings in the
  README diff at `codex-rs/app-server/README.md:619`.
- `codex-rs/config/src/config_toml.rs:186-193` — `local_image_max_dimension`
  carries `#[schemars(range(min = 1))]`; good — prevents zero/negative
  width values from validating.
- `codex-rs/core/src/config/mod.rs:478` — `Config.local_image:
  LocalImageConfig` is grouped at the right level. The four
  `config_tests.rs` precedence fixtures (`config_tests.rs:6431, 6634, 6791,
  6933`) are all updated to set `local_image: LocalImageConfig::default()`
  consistently — no fixture drift.
- `codex-rs/core/config.schema.json:1070-1076` and `:4154-4170` — schema
  emits `LocalImageResizePolicy` as an enum and exposes both fields
  top-level. Good — keeps schema generation deterministic.
- The README addition at `codex-rs/app-server/README.md:619` is a single
  sentence; consider noting that `local_image_max_dimension` is ignored
  when policy is `original`, otherwise a reader might think they need to
  unset both.
- Nit: `LocalImageResizePolicy::ResizeToFit` is `#[default]`, matching the
  existing `PromptImageMode` default of resizing — backward-compatible.
  Worth a one-line CHANGELOG entry since this surfaces a knob users can
  now break themselves with.

## Verdict

`merge-after-nits` — README clarification + CHANGELOG line are the only
asks. Schema and test wiring are correct.
