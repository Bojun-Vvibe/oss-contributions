---
pr: openai/codex#20245
sha: e8d8b818fc533d4b9139d3dc96a536e42ad680fa
verdict: merge-after-nits
reviewed_at: 2026-04-30T00:00:00Z
---

# [Codex] Add browser use external feature flag

URL: https://github.com/openai/codex/pull/20245
Files: `codex-rs/core/config.schema.json`, `codex-rs/features/src/lib.rs`, `codex-rs/features/src/tests.rs`
Diff: 23+/0-

## Context

Codex already has a `BrowserUse` feature gate
(`features/src/lib.rs:169`, key `browser_use`, Stage::Stable). This PR adds
a sibling `BrowserUseExternal` flag for "Browser Use integration with
external browsers" — separating "we ship a browser-use surface" from
"we let users point that surface at an externally-managed browser."

## What's good

- Adds `Feature::BrowserUseExternal` enum variant
  (`features/src/lib.rs:170-174`) with the same "Requirements-only gate:
  this should be set from requirements, not user config" doc comment as
  `BrowserUse` — keeps the per-account-policy contract uniform.
- Registers the spec at `features/src/lib.rs:903-908` with
  `stage: Stage::Stable, default_enabled: true` matching the parent
  `BrowserUse` defaults — so an account that enables `BrowserUse` doesn't
  silently lose the external-browser path.
- Schema additions at `config.schema.json:361-363` and `:3306-3308` cover
  both flag locations (top-level features map and per-profile override
  block) — preventing the "set the flag, get a schema-validation error"
  trap.
- Test coverage at `features/src/tests.rs:160-166` asserts the trio
  (stage, default_enabled, key→Feature lookup), matching the existing
  pattern in `browser_controls_are_stable_and_enabled_by_default`.

## Nits

- The doc comment at `lib.rs:172-173` says "Allow Browser Use integration
  with external browsers" but never defines what "external" means in
  contrast to the parent `BrowserUse` flag. A one-sentence "vs `BrowserUse`,
  which controls the embedded …" would make the requirements-side gate
  policy decisions easier to reason about.
- The new test cases are inlined into `browser_controls_are_stable_and_enabled_by_default`
  (`tests.rs:158`) which now mixes assertions about three different
  features in one fn. Worth splitting into a helper
  `assert_stable_default_on(Feature, &str)` for readability.
- `default_enabled: true` for an external-integration flag deserves a
  one-line justification in either the PR body or the spec — most
  external-integration gates default to off until explicitly opted into,
  and the asymmetry is worth documenting.
- No CHANGELOG / release-notes entry — given this is a Stage::Stable flag
  shipping enabled by default, downstream consumers parsing the feature
  list will see a new key on next release with no announcement.

## Verdict reasoning

Mechanically correct addition matching the existing `BrowserUse` pattern,
schema is in sync, tests cover the lookup contract. Open questions are
about *meaning* (what does "external" actually gate?) and rollout
defaults, not correctness — hence merge-after-nits rather than
needs-discussion.
