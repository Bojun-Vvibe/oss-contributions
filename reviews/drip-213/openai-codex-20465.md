# Review: openai/codex#20465 — config tests: derive permission profiles directly

- PR: https://github.com/openai/codex/pull/20465
- Head SHA: `b43b0ebcf0650150c50dc0d2ac9bb917110a365c`
- Files: `codex-rs/core/src/config/config_tests.rs` (+25/-41)
- Sapling stack: parent of #20466/#20467 (#20467 reviewed in this drip)
- Verdict: **merge-as-is**

## What it does

Replaces the test-only helper `derive_legacy_sandbox_policy_for_test(...)` with
`derive_permission_profile_for_test(...)` and migrates the four call sites that used it.
Where the helper previously called `derive_permission_profile(...).await` and then immediately
projected the result back to `SandboxPolicy` via `to_legacy_sandbox_policy(Path::new("/"))`
(with a fallback warning + `new_read_only_policy()` on projection failure), it now returns
the `PermissionProfile` directly.

## Notable changes

- `config_tests.rs:139-160` — helper signature changes from `-> SandboxPolicy` to
  `-> PermissionProfile`. Body shrinks from a `derive` + `to_legacy_sandbox_policy` + warning-fallback
  sandwich to a single `cfg.derive_permission_profile(...).await`. The deleted `tracing::warn!(...)`
  branch was load-bearing for the prior projection: any profile that had no legacy-equivalent
  was silently downgraded to read-only and the test would then assert against read-only
  rather than the actual derived state. Removing the projection removes that silent-downgrade
  failure mode.
- `config_tests.rs:2132-2152` (`test_sandbox_config_parsing` block 1, `sandbox_full_access`) —
  `assert_eq!(resolution, SandboxPolicy::DangerFullAccess)` becomes
  `assert_eq!(resolution, PermissionProfile::Disabled)`. The two represent the same operational
  state (full access / no sandbox).
- `config_tests.rs:2153-2163` (block 2, `sandbox_read_only`) — `assert_eq!(resolution, SandboxPolicy::new_read_only_policy())`
  → `assert_eq!(resolution, PermissionProfile::read_only())`. Direct equivalent.
- `config_tests.rs:2185-2218` (block 3, `sandbox_workspace_write` with `writable_root`) —
  the `else` arm changes from a struct-literal `SandboxPolicy::WorkspaceWrite { writable_roots: vec![..], network_access: false, exclude_tmpdir_env_var: true, exclude_slash_tmp: true }`
  to a constructor call `PermissionProfile::workspace_write_with(std::slice::from_ref(&writable_root), NetworkSandboxPolicy::Restricted, true, true)`.
  Note `network_access: false` ↔ `NetworkSandboxPolicy::Restricted` — semantically equivalent
  through the projection. The constructor takes a slice (`std::slice::from_ref` avoids
  allocating a `Vec` for a single element) which is a minor allocation win.
- `config_tests.rs:2225-2245` (block 4, second `sandbox_workspace_write`) — same shape as
  block 3, uses `&[writable_root]` array literal instead of `from_ref` (the local `writable_root`
  is owned here vs borrowed in block 3, hence the difference).
- `config_tests.rs:7146-7180` `test_untrusted_project_gets_workspace_write_sandbox` — windows arm
  becomes `assert_eq!(resolution, PermissionProfile::read_only())`, unix arm becomes
  `assert_eq!(resolution, PermissionProfile::workspace_write())`. Tightens both arms from
  `matches!` patterns to exact equality.
- `config_tests.rs:7184` and `:7250` — two test functions are renamed from
  `derive_sandbox_policy_*` to `derive_permission_profile_*` to match the new canonical model;
  function bodies are updated to assert on `PermissionProfile` directly.

## Reasoning

This is the foundational PR for the assertion-migration work that #20466 / #20467 build on.
By changing the helper return type, the compiler forces every call site to migrate; without
this change, callers could continue to add `legacy_sandbox_policy()` assertions and the
canonical/projection drift would not be caught.

The deleted projection-fallback branch (the `tracing::warn!` arm) is the most subtle win:
under the old helper, any `PermissionProfile` that did not round-trip through
`to_legacy_sandbox_policy(...)` cleanly was silently downgraded to `new_read_only_policy()`,
which meant tests asserting `SandboxPolicy::new_read_only_policy()` would pass for both
"correctly derived read-only" and "couldn't project, fell back to read-only" — two distinct
outcomes that should be asserted distinctly. The new helper makes the second case
impossible (it just returns the profile, no projection involved), so the migrated assertions
unambiguously test the derivation path.

Two function renames (`derive_sandbox_policy_*` → `derive_permission_profile_*`) keep the
test names aligned with what they actually exercise post-migration; this is a good habit
because grep-by-function-name is the typical way to find tests for a given subsystem.

The verification list cites the migrated tests by name plus a `just fix -p codex-core`
lint pass.

## Nits

None. Same shape as #20467, equally surgical, equally correct. Merge as-is.
