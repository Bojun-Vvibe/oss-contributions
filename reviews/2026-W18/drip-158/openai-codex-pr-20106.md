# openai/codex#20106 — linux-sandbox: switch helper plumbing to PermissionProfile

- PR: https://github.com/openai/codex/pull/20106
- Head SHA: `b2dfa64af7e5bb607236e113459793ba7c7b20a4`
- Author: bolinfest
- Diff: +199/-517 across 11 files (cli/debug_sandbox.rs, core/landlock.rs, linux-sandbox/{bwrap,landlock,linux_run_main,linux_run_main_tests}.rs, linux-sandbox/tests/{landlock,managed_proxy}.rs, sandboxing/{landlock,landlock_tests,manager}.rs)

## What changed

Collapses the helper's parallel-policy plumbing into a single canonical `PermissionProfile` input. The CLI flag `--sandbox-policy` plus its split companions `--file-system-sandbox-policy` and `--network-sandbox-policy` (`linux-sandbox/src/linux_run_main.rs:41-65` previously) are deleted in favor of a single `--permission-profile` parsed via `parse_permission_profile`. The helper-side resolver `resolve_sandbox_policies(...)` and its 4-variant `ResolveSandboxPoliciesError` enum (~140 lines including `PartialSplitPolicies`, `SplitPoliciesRequireDirectRuntimeEnforcement`, `FailedToDeriveLegacyPolicy`, `MismatchedLegacyPolicy { provided, derived }`) are replaced by a one-line `resolve_permission_profile(...)` whose only failure is `MissingConfiguration`.

Call sites collapse correspondingly: `apply_sandbox_policy_to_current_thread(&sandbox_policy, network_sandbox_policy, ...)` becomes `apply_permission_profile_to_current_thread(&permission_profile, ...)`, with the function deriving `(file_system_sandbox_policy, network_sandbox_policy) = permission_profile.to_runtime_permissions()` internally (`linux-sandbox/src/landlock.rs:46-50`). The exported helper API in `sandboxing/src/landlock.rs` renames `create_linux_sandbox_command_args_for_policies` → `create_linux_sandbox_command_args_for_permission_profile` and propagates that through both `core/src/landlock.rs:38` and `cli/src/debug_sandbox.rs:222`.

Test fixtures collapse the structural `SandboxPolicy::WorkspaceWrite { writable_roots, network_access, exclude_tmpdir_env_var, exclude_slash_tmp }` literal into the new constructor `FileSystemSandboxPolicy::workspace_write(&[...], exclude_tmpdir_env_var, exclude_slash_tmp)` (e.g. `linux-sandbox/src/bwrap.rs:1066,1085,1399,1532`), and the `DangerFullAccess` projection becomes `FileSystemSandboxPolicy::unrestricted()`.

## Observations

The doc comment update at `core/src/landlock.rs:18-21` honestly captures the new contract — "we pass the canonical permission profile as JSON and let the helper derive the runtime filesystem/network policies" — and that derivation moves from caller to helper, which is the right direction for a security boundary (one canonical input, no caller-side disagreement to validate). The deleted `MismatchedLegacyPolicy { provided, derived }` arm is the nicest evidence: previously the helper had to reject inputs where the legacy projection didn't match the split policies, which is a class of bug that simply cannot exist in the new shape.

The `compatibility_sandbox_policy_for_permission_profile` import is dropped at `core/src/landlock.rs:6`, so any downstream that depended on the legacy projection being computed at the call site needs to re-derive it locally — worth grepping the rest of the workspace before merge to confirm `compatibility_sandbox_policy_for_permission_profile` has no remaining external consumers, since this PR likely also drops or weakens that helper.

One concrete net-deletion to flag: the `apply_landlock_fs && !sandbox_policy.has_full_disk_write_access()` predicate at `linux-sandbox/src/landlock.rs:61,71` flips to `file_system_sandbox_policy.has_full_disk_write_access()`. Since `file_system_sandbox_policy` is now derived from `permission_profile.to_runtime_permissions()`, the `WorkspaceWrite { writable_roots: empty }` corner case that used to short-circuit through `SandboxPolicy::has_full_disk_write_access` needs a test pinning that the derived `FileSystemSandboxPolicy` returns the same boolean — the diff removes those direct-shape tests (`SandboxPolicy::WorkspaceWrite { ... }` literals) but I don't see a replacement that asserts the `permission_profile → file_system_sandbox_policy` derivation produces the same `has_full_disk_write_access()` answer for each pre-existing fixture.

## Verdict

`merge-after-nits`
