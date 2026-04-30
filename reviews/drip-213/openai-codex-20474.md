# Review: openai/codex#20474 — [plugin] Add Canva to suggesteable list.

- PR: https://github.com/openai/codex/pull/20474
- Head SHA: `b1f32e38551af48e107385e8ade72295df75bbf1`
- Files: `codex-rs/core-plugins/src/lib.rs` (+1/-0)
- Verdict: **merge-as-is**

## What it does

One-line addition of `"canva@openai-curated"` to the `TOOL_SUGGEST_DISCOVERABLE_PLUGIN_ALLOWLIST`
constant in `codex-rs/core-plugins/src/lib.rs:25`.

## Notable changes

- `lib.rs:25` — inserts `"canva@openai-curated",` between `"google-drive@openai-curated"` and
  `"teams@openai-curated"`. The list contains 8 entries pre-change (`gmail`, `google-calendar`,
  `google-drive`, `teams`, `sharepoint`, `outlook-email`, etc., all in the `@openai-curated`
  namespace); this brings it to 9.

## Reasoning

This is a configuration/data change with zero behavioural risk. The constant is a `&[&str]`
allowlist consumed by the plugin-suggestion subsystem; appending one entry adds one allowed
suggestion source. There is no precedence ordering, no version pin, no test surface to wire,
and no migration. The naming follows the existing convention exactly (`<service>@openai-curated`).
The only possible concern is whether the `canva@openai-curated` plugin actually exists and is
ready for discovery, but that is a release-coordination question outside the diff.

The PR body confirms the intent (`Add Canva to suggesteable list`) and the diff matches.

## Nits

None. Merge as-is.
