---
pr_number: 19394
repo: openai/codex
head_sha: 1bdc3bd56d81c0b6dc21d2644a2797154d17baf7
verdict: merge-after-nits
date: 2026-04-25
---

# openai/codex#19394 — remove core legacy `SandboxPolicy` round-trips

**What changed.** 13 files / +118 / −348. Across `core/src/exec_policy.rs`, `core/src/session/turn_context.rs`, `core/src/tools/handlers/shell.rs`, `core/src/tools/runtimes/shell/unix_escalation.rs`, `core/src/unified_exec/process_manager.rs`, and `sandboxing/src/policy_transforms.rs`, every `&SandboxPolicy` parameter on the hot path is replaced by `&PermissionProfile` (or `PermissionProfile` by-value where small). `ExecApprovalRequest` (line 204) gains `permission_profile: PermissionProfile` and drops `sandbox_policy: &SandboxPolicy`. `render_decision_for_unmatched_command` (line 580) takes the profile directly. `web_search_mode_for_turn` (config/mod.rs line 1583) is rewritten to match on `PermissionProfile::Disabled` instead of `SandboxPolicy::DangerFullAccess`.

**Why it matters.** The previous code went `PermissionProfile → SandboxPolicy → effective FS/network perms`, and per the body, that round trip was lossy for split filesystem policies. Profiles are now the canonical runtime source of truth (set up in #19391), so the legacy projection layer is dead weight here.

**Concerns.**
1. **Windows-readonly special-case moved into a helper.** `profile_is_managed_read_only` (exec_policy.rs line 670) replaces the inline `cfg!(windows) && matches!(sandbox_policy, SandboxPolicy::ReadOnly { .. })`. The new check requires `PermissionProfile::Managed { .. }` AND `FileSystemSandboxKind::Restricted` AND `get_writable_roots_with_cwd(Path::new("/")).is_empty()`. The third clause is the suspicious one — a profile that is managed-restricted with one writable root under `/tmp` will return `false`, whereas the legacy `SandboxPolicy::ReadOnly` ignored writable_roots at this branch. Confirm parity with a test case: `Managed { .. }` + restricted FS + writable `/tmp/work` should NOT be treated as "lacks sandbox protections". The diff doesn't add such a test.
2. **`PermissionProfile::Disabled | PermissionProfile::External { .. }` substitution at line 605** preserves the prior "explicitly disabled → allow without approval under `Never`" semantics. Looks right, but worth a single comment line tying these two variants to the prior `DangerFullAccess | ExternalSandbox` pair so future readers don't redo the matching exercise.
3. **`from_legacy_sandbox_policy` is the bridge.** Tests use `permission_profile_from_sandbox_policy` (exec_policy_tests.rs line 120) as a thin wrapper around `PermissionProfile::from_legacy_sandbox_policy(&SandboxPolicy::new_read_only_policy())`. Once #19395 lands, that helper should be deleted — leaving it in place tempts future callers to keep round-tripping.
4. Stack PR (sits between #19393 and #19395). Don't land standalone; merge in stack order.

Solid net-negative diff doing exactly what the body claims. Land after a parity test for the Windows managed-readonly branch.
