# Review: openai/codex#20467 — config tests: assert sandbox-mode fallbacks via profiles

- PR: https://github.com/openai/codex/pull/20467
- Head SHA: `110d05accfb35cb93bea3dabce00a2a1bd95f841`
- Files: `codex-rs/core/src/config/config_tests.rs` (+22/-14)
- Sapling stack: child of #20465 (drip-213), parent of #20468/#20469 (covered prior drips for sibling PRs)
- Verdict: **merge-as-is**

## What it does

Sapling-stack continuation of the "stop asserting through `legacy_sandbox_policy()`" cleanup
applied across `codex-core/src/config/config_tests.rs`. Migrates four test functions from
asserting on the legacy `SandboxPolicy` projection to asserting on the canonical
`PermissionProfile` / `permissions.{file_system,network}_sandbox_policy()` runtime state that
the config loader now produces directly.

## Notable changes

- `config_tests.rs:3174-3180` `profile_sandbox_mode_overrides_base` — replaces
  `assert!(matches!(&config.legacy_sandbox_policy(), &SandboxPolicy::DangerFullAccess))` with
  `assert_eq!(config.permissions.permission_profile(), PermissionProfile::Disabled)`. Tightens
  from a `matches!` pattern (which would accept any `DangerFullAccess` shape) to exact equality
  on the canonical enum.
- `config_tests.rs:3208-3225` `cli_override_takes_precedence_over_profile_sandbox_mode` — splits
  the unix arm into two assertions: a positive `can_write_path_with_cwd(cwd, cwd)` check on the
  derived filesystem policy AND an `assert_eq!(...network_sandbox_policy(), NetworkSandboxPolicy::Restricted)`
  on the network policy. Stronger than the prior `matches!(SandboxPolicy::WorkspaceWrite { .. })`
  because the `..` would have accepted any combination of `network_access` / `exclude_tmpdir_env_var`
  / `writable_roots`; the new pair pins the network restriction explicitly.
- `config_tests.rs:7507-7525` `test_untrusted_project_gets_unless_trusted_approval_policy` — same
  shape: windows arm now asserts `permission_profile() == read_only()`, unix arm asserts
  `can_write_path_with_cwd` + `network_sandbox_policy() == Restricted`. Same strengthening.
- `config_tests.rs:7545-7548` `requirements_disallowing_default_sandbox_falls_back_to_required_default`
  and `:7587-7590` `explicit_sandbox_mode_falls_back_when_disallowed_by_requirements` — both swap
  `assert_eq!(legacy_sandbox_policy(), SandboxPolicy::new_read_only_policy())` for
  `assert_eq!(permission_profile(), PermissionProfile::read_only())`. These are direct equivalents
  through the projection, with no behavioural change.

## Reasoning

This is the right continuation of the migration: the prior PRs in the stack moved the loader to
produce `PermissionProfile` as the source of truth and made `SandboxPolicy` a downstream
projection. Tests that assert on the projection rather than the canonical state are at risk of
silently passing when the projection changes shape but the canonical state regresses. Migrating
each test to assert on `permission_profile()` directly closes that gap.

The migration is also surgical — the PR description explicitly preserves legacy assertions in
tests that are *about* the compatibility projection (`sandbox_workspace_write` writable-roots tests,
etc.), which is the right call.

The unix-arm strengthening (positive `can_write_path_with_cwd` + explicit `network_sandbox_policy`
equality) is a non-trivial improvement: the prior `matches!(SandboxPolicy::WorkspaceWrite { .. })`
would have passed even if a future refactor accidentally set `network_access: true`, because the
network field was hidden inside the `..`. The new shape pins both fields independently.

Verification commands listed in the PR body cover all four touched tests by name plus a
`just fix -p codex-core` lint pass.

## Nits

None. This is exactly the right shape for a "migrate test assertions to canonical model" PR
in the middle of a 70+-PR Sapling stack. Merge as-is.
