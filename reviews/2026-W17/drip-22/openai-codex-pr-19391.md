# PR #19391 — permissions: make runtime config profile-backed

- URL: https://github.com/openai/codex/pull/19391
- Author: bolinfest
- Head SHA: `90bf5686eb17b09da76ac2d10989c33326ce4e2a`

## Summary

Promotes `PermissionProfile` to the canonical runtime permissions shape inside
`Permissions`, alongside the legacy `sandbox_policy` / `file_system_sandbox_policy` /
`network_sandbox_policy` projections that compatibility callers still need. Adds
`Permissions::set_legacy_sandbox_policy` and `Permissions::set_permission_profile`
mutators that keep all four fields in lock-step, and threads the profile through
`build_exec_request`, `PermissionsInstructions::from_permission_profile`, and the
app-server exec path so the runtime no longer has to re-derive enforcement from a
lossy `SandboxPolicy`.

## Specific callouts

- `codex-rs/core/src/config/mod.rs:191-260` — `Permissions` now holds
  `permission_profile: Constrained<PermissionProfile>` as the canonical field; the
  three projections are explicitly tagged "Legacy/Runtime projection retained
  while callers migrate". The doc comments here are the load-bearing contract for
  future cleanup PRs in the stack (#19392/#19393/#19394/#19395). Worth pinning a
  matching issue/tracker so the projections actually get removed and don't
  ossify.
- `codex-rs/core/src/config/mod.rs:235-303` — `set_legacy_sandbox_policy` and
  `set_permission_profile` both call `can_set` on **both** constraints before
  mutating either. Good — this avoids the half-applied state where
  `permission_profile` accepts but `sandbox_policy` would have rejected. The
  derivations (`from_legacy_sandbox_policy_for_cwd`,
  `compatibility_sandbox_policy_for_permission_profile`) are the only round-trip
  surface; if those two ever drift from each other, every mutation here silently
  corrupts the projection. A round-trip property test
  (`profile -> legacy -> profile == profile` for the constrained subset) is the
  single highest-value addition.
- `codex-rs/core/src/config/mod.rs:309-329` — `constrained_permission_profile_from_sandbox_projection`
  layers a constraint on `PermissionProfile` derived from
  `Constrained<SandboxPolicy>`. The closure calls `to_legacy_sandbox_policy`
  inside the validator — this means every candidate profile is converted on the
  hot path. Cheap today, but if the conversion ever allocates non-trivially
  (e.g. resolves writable roots), it'll show up. Consider memoising or, better,
  deriving the constraint from the profile shape directly so the closure can
  stay infallible.
- `codex-rs/app-server/src/codex_message_processor.rs:2212-2295` — The
  three-arm `if let Some(permission_profile)` is now collapsed into a single
  `effective_permission_profile` value. The middle arm (legacy `sandbox_policy`
  fallback path, ~line 2263-2275) reconstructs the profile via
  `from_runtime_permissions_with_enforcement` + `SandboxEnforcement::from_legacy_sandbox_policy`.
  That reconstruction is identical to what `Permissions::permission_profile()`
  used to do — please call the helper there instead of inlining, otherwise
  these two sites will drift the next time enforcement gains a variant.
- `codex-rs/core/src/context/permissions_instructions.rs:62-118` — New
  `from_permission_profile` becomes the primary builder; legacy `from_policy`
  delegates via `PermissionProfile::from_legacy_sandbox_policy(sandbox_policy)`.
  `sandbox_prompt_from_profile` (lines 137-158) handles `Disabled` / `External`
  identically (`SandboxMode::DangerFullAccess`) — that's a behavior change vs.
  the old `SandboxPolicy::ExternalSandbox` arm, which also returned
  `DangerFullAccess` but did **not** consult writable-root logic. Looks
  intentional, but worth one explicit test asserting the prompt for
  `External { .. }` matches the prompt for `Disabled` so future refactors don't
  silently diverge.
- `codex-rs/core/src/config/config_tests.rs:5201-5731` — Four `Permissions {
  ... }` literals get updated with the new `permission_profile` field. Each one
  hardcodes `PermissionProfile::from_legacy_sandbox_policy(&SandboxPolicy::new_read_only_policy())`.
  Consider a small `Permissions::with_legacy_sandbox_policy(SandboxPolicy)`
  test helper — five identical six-line blocks today, will be ten by the time
  the rest of the stack lands.
- `codex-rs/core/src/config/config_tests.rs:6531-6566` — New
  `permission_profile_override_falls_back_when_disallowed_by_requirements`
  asserts that the requirements-driven fallback rebuilds **both**
  `sandbox_policy` and `permission_profile`. Good coverage of the constraint
  interaction; this is the test that would have caught the half-applied state
  if the mutators above lacked the dual `can_set` guard.

## Risks

- Two sources of truth (`permission_profile` and the three legacy projections)
  during the migration window. The mutators enforce coherence on writes, but
  any code path that bypasses them and writes a field directly (e.g. via
  `permissions.sandbox_policy.set(...)` from outside this module) will desync.
  The `pub` visibility of the projection fields makes this possible. Either
  narrow them to `pub(crate)` or add a `#[doc(hidden)]` warning.
- `compatibility_sandbox_policy_for_permission_profile` is now load-bearing for
  every legacy consumer. Its behavior on `External { .. }` profiles isn't
  visible in this diff — needs a paired test in `codex-sandboxing` so the
  contract is explicit.

## Verdict

**Verdict:** merge-after-nits

Strong refactor with a coherent migration story and good dual-write discipline
in the mutators. Two cleanups before merge: (1) call
`Permissions::permission_profile()` from the legacy fallback in
`codex_message_processor.rs` instead of inlining the reconstruction, and (2)
add a round-trip property test for `set_permission_profile` ↔
`set_legacy_sandbox_policy` so the four fields can't silently drift.
