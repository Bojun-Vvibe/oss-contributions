# openai/codex PR #19728 — config: re-export NetworkConstraints/RequirementsToml from codex_config

- **PR**: https://github.com/openai/codex/pull/19728
- **HEAD short**: from `gh pr diff` body — touches only `codex-rs/core/src/config/config_tests.rs`
- **Touch**: 1 file, +4/−4 (paths in test references)

## Context

Two test setup blocks in `config_tests.rs` constructed
`NetworkConstraints`, `NetworkRequirementsToml`, `ConfigRequirementsToml`,
and `SandboxModeRequirement` via `crate::config_loader::...` paths. As
part of the ongoing carve-out from monolithic `codex-core` into the
`codex-config` crate (visible in earlier review activity around the
`config_loader → codex-config` migration), those types now live behind
`codex_config::*` re-exports. The intra-crate path `crate::config_loader`
was working only by virtue of an existing `pub use` shim that the parent
PR is presumably about to drop.

## Diff

Four call sites flipped:

- `config_tests.rs:917` — `crate::config_loader::NetworkConstraints` →
  `codex_config::NetworkConstraints`
- `config_tests.rs:923` — `crate::config_loader::NetworkRequirementsToml`
  → `codex_config::NetworkRequirementsToml`
- `config_tests.rs:6748` — `crate::config_loader::ConfigRequirementsToml`
  → `codex_config::ConfigRequirementsToml`
- `config_tests.rs:6749` — `crate::config_loader::SandboxModeRequirement`
  → `codex_config::SandboxModeRequirement`

Pure path rewrite; no behavior, no semantic change. The two test
functions (`managed_unrestricted_permission_profile_still_enables_network_requirement`
and `permission_profile_override_falls_back_when_disallowed_by_requirements`)
keep their existing setup vectors and assertions.

## Risks / nits

- Verify the `codex_config` crate is actually in `[dev-dependencies]` for
  `codex-core` (or already in `[dependencies]`); the PR diff doesn't show
  Cargo.toml so this is taken on faith. If the parent of this PR is the
  one that adds the dependency, this PR must merge after.
- The remaining `crate::config_loader::*` references in the same file —
  if any — should be flipped in the same commit so the eventual removal
  of the `pub use` shim is atomic. A grep over `config_tests.rs` for
  `config_loader::` after this PR lands would confirm.
- This is a textbook "tag-along test path migration" — the kind of PR
  that should land same-day as the dependent shim removal so the
  intermediate state isn't broken on `main`.
- Naming: `codex_config::NetworkConstraints` and the
  `codex::config_loader::NetworkConstraints` shim should not coexist
  long-term — one read-side definition, one re-export, period. Whoever
  owns the carve-out should add a `#[deprecated]` to the crate-internal
  shim to prevent accidental future use.

## Verdict

Verdict: merge-as-is
