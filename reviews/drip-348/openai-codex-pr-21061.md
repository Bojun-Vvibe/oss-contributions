# openai/codex#21061 — [codex] support managed app tool approval requirements

- **Head SHA**: `aa6040320366`
- **Verdict**: `merge-after-nits`

## Summary

Extends managed (Cloud/MDM) requirements to allow per-tool approval-mode overrides under an existing app entry. Adds new TOML types `AppToolRequirementToml` and `AppToolsRequirementsToml` in `codex-rs/config/src/config_requirements.rs` and threads `tools` through `AppRequirementToml`. Includes a parsing test in `codex-rs/cloud-requirements/src/lib.rs`. +401/-20.

## Findings

- `codex-rs/config/src/config_requirements.rs:81-91` — `AppToolsRequirementsToml` uses `#[serde(default, flatten)]` on `tools: BTreeMap<String, AppToolRequirementToml>`. Combined with the parent struct having only this single field, the flatten lets users write `[apps.<id>.tools."calendar/list_events"]` directly. The test at `cloud-requirements/src/lib.rs:1413-1438` confirms this shape works. Good. But: with `flatten`, any unknown key under `tools` becomes a (key, empty struct) entry rather than a deserialization error. If MDM admins typo `aproval_mode`, the field silently becomes `None` and the entire requirement is a no-op. Consider `#[serde(deny_unknown_fields)]` on `AppToolRequirementToml` and a follow-up validator that warns on `is_empty()` per-tool entries.
- `config_requirements.rs:99-107` — `AppRequirementToml::is_empty` short-circuits when both `enabled` and `tools` are absent/empty. The chained `.is_none_or(AppToolsRequirementsToml::is_empty)` is correct but mildly hard to read; an explicit `match` reads better and matches the rest of the file's style.
- `config_requirements.rs:115` — `AppsRequirementsToml::is_empty` switches from `app.enabled.is_none()` to `AppRequirementToml::is_empty`, which now also requires tools to be empty. This is the behavior change that lets a `tools`-only override survive precedence merging. Worth a one-line comment explaining the semantic shift, since the previous form looked like dead code being tidied up.
- `cloud-requirements/src/lib.rs:1426` — the test only covers a single-tool case. Add at least one case asserting that `[apps.X.tools."a"]` and `[apps.X.tools."b"]` both round-trip into the same `AppToolsRequirementsToml.tools` map, since the `flatten` semantics are non-obvious.
- Naming: `AppToolApproval` (imported from `mcp_types`) is the same enum used elsewhere for runtime approval decisions. Reusing it here is correct and avoids drift, but means any future variant added for runtime UX (e.g. an "ask once per session" mode) silently becomes valid in MDM TOML. Worth a `// reused from mcp_types` comment near the import at `:21`.

## Recommendation

Mechanically clean and the test demonstrates intent. Tighten with `deny_unknown_fields` on the per-tool struct (or a runtime warning) so MDM typos don't silently no-op, expand the parse test to cover multiple tools, and add a one-line comment on the `AppsRequirementsToml::is_empty` semantic change. Then ship.
