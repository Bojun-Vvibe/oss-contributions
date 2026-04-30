# openai/codex #20398 — protocol: drop cwd-less legacy profile constructor

- Head SHA: `37aa2f81571d355ea6df5b627ce32f9d680c3440`
- Files: `codex-rs/core/src/session/tests.rs`, `codex-rs/protocol/src/models.rs`
- Size: +11 / -33

## What it does

Mid-stack PR (~position 11 of a 25-PR Sapling stack) that removes the
`PermissionProfile::from_legacy_sandbox_policy(&SandboxPolicy)` constructor
from `codex-rs/protocol/src/models.rs:478-484`. The cwd-bearing sibling
`from_legacy_sandbox_policy_for_cwd(&SandboxPolicy, &Path)` is intentionally
left intact a few lines below (`models.rs:486-...`), which is the right call:
workspace-write semantics are only well-defined relative to a project root,
and the removed constructor was a foot-gun that hid that requirement.

## What's removed

The deletion is small and surgical:
1. The function body at `models.rs:478-484` (5 lines) — pure dispatch into
   `from_runtime_permissions_with_enforcement` with the cwd-less filesystem
   policy.
2. The unit test `permission_profile_presets_match_legacy_defaults`
   (`models.rs:1832-1841`) which tested the now-removed constructor against
   the read-only and workspace-write presets. That test had no other
   assertion value once the function it tested is gone.
3. The body of `permission_profile_round_trip_preserves_disabled_sandbox`
   (`models.rs:1844-...`) is rewritten to use `PermissionProfile::Disabled`
   directly instead of round-tripping through `SandboxPolicy::DangerFullAccess`.

## What's migrated

Two surviving call sites (the only ones left on this stack) move to direct
variant construction:

1. `codex-rs/protocol/src/models.rs:1916-1918` — the external-sandbox round
   trip test now constructs `PermissionProfile::External { network:
   NetworkSandboxPolicy::Restricted }` literally, matching the
   `SandboxPolicy::ExternalSandbox { network_access: Restricted }` it is
   testing against.
2. `codex-rs/core/src/session/tests.rs:1591-1601` — the
   `session_configured_reports_permission_profile_for_external_sandbox`
   integration test takes `expected_permission_profile = External {network:
   Restricted}` as the source of truth, clones it into the
   `with_config(...)` closure, and the post-build assertion at
   `tests.rs:1610` checks `test.session_configured.permission_profile ==
   expected_permission_profile`. The `set_legacy_sandbox_policy(sandbox_policy)`
   call is preserved so the legacy enforcement plumbing remains under test.

## Assessment

This is a textbook deletion-only migration: one suspicious-by-design API
removed, two callers transitioned to the explicit variant they always
should have used, one obsolete unit test pruned, the parallel cwd-bearing
constructor preserved for the cases that legitimately need it. The net -22
lines is real cleanup, not a behavioral change.

The Sapling stack metadata in the body shows this depends on multiple
sibling PRs that migrate the other call sites; the assumption is that the
parent PRs land first so this leaf doesn't become a build break in
isolation. Confirmed by `cargo test -p codex-protocol` and the named
session test at the cited line.

Verdict: merge-as-is
