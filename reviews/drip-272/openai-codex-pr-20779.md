# openai/codex PR #20779 — feat: generic hints for configs

- URL: https://github.com/openai/codex/pull/20779
- Head SHA: `d2aa46354562d9830f681c2a16a8abac722b962a`
- Verdict: **merge-after-nits**

## What it does

Refactors the JSON-schema generation for feature toggles in
`codex-rs/config/src/schema.rs` so that every feature flag is uniformly typed
as `FeatureToml<NoExtraFeatureConfigToml>` instead of plain `bool`. Today the
generator special-cases `apps_mcp_path_override` (which carries an extra
`{enabled, path}` payload via `AppsMcpPathOverrideConfigToml`) and emits
`bool` for everything else. After this PR, every feature uniformly gets a
`FeatureToml<...>` wrapper with `NoExtraFeatureConfigToml` for features that
don't have extra config.

## Specifics I checked

- `codex-rs/config/src/schema.rs:45-58` swaps both branches of the
  `features_schema` builder (current features + legacy keys) from
  `subschema_for::<bool>()` to
  `subschema_for::<codex_features::FeatureToml<NoExtraFeatureConfigToml>>()`.
- Regenerated `codex-rs/core/config.schema.json` shows ~80 feature properties
  flipping from `"type": "boolean"` to
  `"$ref": "#/definitions/FeatureToml_for_NoExtraFeatureConfigToml"`. The
  previously-special `AppsMcpPathOverrideConfigToml` entry is removed because
  the special case is now subsumed by the generic mechanism.

## What I like

- Eliminates a special case. Adding the next "feature with extra config"
  becomes a one-line `FeatureToml<MyFeatureConfigToml>` substitution rather
  than another bespoke branch in `schema.rs`.
- The schema now expresses the semantic that a feature is more than a bare
  bool (it can carry hints/extra fields), even when it currently has none.

## Concerns / nits

1. **Backward compatibility of `bool` config values.** TOML configs in the
   wild today look like `[features]\napps = true`. Verify
   `FeatureToml<NoExtraFeatureConfigToml>` deserializes a bare `true`/`false`
   exactly the way the previous `bool` did. If the new wrapper also requires
   `apps = { enabled = true }` form, that's a silent breaking change for
   every user. The PR doesn't add a regression test for this, and given how
   prominent the feature flag table is, this needs an explicit
   `serde(untagged)` / `From<bool>` test before merge.
2. **Schema consumers.** Any external tooling that introspects this schema
   (IDE TOML LSPs, internal dashboards) will see the property type flip from
   `boolean` to `$ref FeatureToml_for_NoExtraFeatureConfigToml`. Worth a
   note in the PR body / CHANGELOG.
3. **PR body is empty.** For a refactor that touches the public config
   schema, a one-paragraph "why" + "what users will/won't see" would speed
   up review materially.

## Verdict rationale

The mechanical change is clean and well-scoped, but item (1) above is a
hard prerequisite — needs either a test demonstrating bool deserialization
still works or an explicit migration note. With that addressed: ship.
