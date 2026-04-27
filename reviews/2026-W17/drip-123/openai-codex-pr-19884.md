# openai/codex#19884 — Add MCP app feature flag

- **PR**: https://github.com/openai/codex/pull/19884
- **Author**: mzeng-openai (Matthew Zeng)
- **Head SHA**: `111edb60cda0a18bfa3f8792234a10cecc8a9832`
- **Verdict**: `merge-as-is`

## Context

The codex feature-flag system at `codex-rs/features/src/lib.rs` is the central registry for incubating capabilities — each entry has an enum variant, a `key` for TOML config, a `Stage` (`UnderDevelopment` / `Beta` / `Stable`), and a `default_enabled` boolean. New incubation work in the MCP-app surface (the broader effort tracked by adjacent PRs #19882 hooks-browser and #19881 deferred-MCP unification) needs a stable feature key to gate visibility before the surface is ready for general exposure. This PR is the minimum-surface registration of that flag — it ships only the registry entry and the JSON-schema reflections, no wired-up consumers yet.

## What it changes

Three coordinated additions: (1) the `Feature::EnableMcpApps` enum variant at `codex-rs/features/src/lib.rs:147-148` placed alphabetically between `Apps` and `ToolSearch`; (2) a `FeatureSpec` entry at `:836-841` with `key: "enable_mcp_apps"`, `stage: Stage::UnderDevelopment`, `default_enabled: false`; and (3) two regenerated additions to `codex-rs/core/config.schema.json` at `:400-411` and `:2607-2618` — both are `"enable_mcp_apps": { "type": "boolean" }` injected into the same alphabetical slot in the two schema sections that mirror the feature surface (the per-project override section and the global config section).

## Design analysis

This is the textbook shape of a "register the flag separately from any consumer" PR, and the reasons are worth naming because they're why this kind of PR should land independently. First, the `Stage::UnderDevelopment + default_enabled: false` pair is the conservative pairing — `UnderDevelopment` is the lowest visibility stage in the codex feature ladder, and `default_enabled: false` means even users on the bleeding-edge build don't see the surface unless they opt in via `[features] enable_mcp_apps = true` in `~/.codex/config.toml`. That's the right safety posture for a flag whose consumers don't exist yet and whose presence should be a no-op for everyone. Second, splitting the registration from the consumer wiring lets the schema regeneration land cleanly without bundling unrelated diffs — the `config.schema.json` deltas are the mechanical output of a `cargo run --bin gen-config-schema` (or equivalent) regen, and reviewers can verify them by inspection: same key inserted in the same alphabetical position in both sections, no other drift. Third, the alphabetical insertion (`enable_fanout` → `enable_mcp_apps` → `enable_request_compression`) is consistent with the existing convention in both files, which keeps merge conflicts with adjacent feature additions to a minimum.

## Risks

Effectively zero. The flag is registered but unread — there is no `is_enabled(Feature::EnableMcpApps)` call anywhere in the diff, so the only behavioral surface is "users can now write `enable_mcp_apps = true` in their config and not get a warning about an unknown key". Any downstream consumer is a separate PR and a separate review. The two schema-section duplications (`:400` and `:2607`) are slightly unusual but match the existing pattern — `enable_fanout` and `enable_request_compression` both also appear in both sections, so this is the established shape and not a redundancy bug. The only failure mode I can construct is "a future PR adds `Feature::EnableMcpApps` to the `is_enabled` check site without verifying the gate is `false` by default", which is a problem for that PR's review, not this one.

## Suggestions

None. This is the right shape and the right scope. If pedantic: the doc-comment on the enum variant at `:147` reads `/// Enable MCP apps.` which is slightly tautological (the variant name and key already say that) — a sentence about *what* MCP apps are or *what* the surface gates would be more useful for future archeology. Not blocking.

## What I learned

The "register flag, then wire consumers in a follow-up" cadence is the right one for any feature flag whose first consumer is more than a few lines — it lets the registration itself be a no-op-by-construction PR (verifiable by grep: zero new `is_enabled` call sites, zero new behavior) and lets the consumer PR focus reviewer attention on the actual logic. Bundling the two together would have made this PR ~10x larger and made it harder to verify the "default-off" property by inspection.
