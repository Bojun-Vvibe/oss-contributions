# Review: openai/codex#20398 — protocol: drop cwd-less legacy profile constructor

- PR: https://github.com/openai/codex/pull/20398
- Author: bolinfest (Michael Bolin)
- headRefOid: `37aa2f81571d355ea6df5b627ce32f9d680c3440`
- Files: `codex-rs/protocol/src/models.rs` (+4/-25), `codex-rs/core/src/session/tests.rs` (+7/-8)
- Verdict: **merge-as-is**

## Analysis

This PR removes `PermissionProfile::from_legacy_sandbox_policy(&SandboxPolicy)` from
`codex-rs/protocol/src/models.rs:475-482` (the cwd-less variant) along with the
`permission_profile_presets_match_legacy_defaults` test that exercised it. The cwd-bearing
`from_legacy_sandbox_policy_for_cwd` remains, which is the correct survivor: `WorkspaceWrite`
semantics are only well-defined relative to a project root, so the cwd-less constructor was a
foot-gun the migration explicitly wanted to retire.

The two remaining call sites (both in tests) are converted to direct enum constructors:
`PermissionProfile::Disabled` for the round-trip-preserves-disabled-sandbox case
(`models.rs:1830-1845`), and `PermissionProfile::External { network: NetworkSandboxPolicy::Restricted }`
for the external-sandbox round-trip (`models.rs:1914-1922`) and the parallel
`session_configured_reports_permission_profile_for_external_sandbox` core test
(`session/tests.rs:1591-1607`). The session test now declares `expected_permission_profile`
once and clones it for the config builder, removing the slightly-spooky pattern of computing
`expected_permission_profile` _after_ `builder.build()` from the original sandbox policy.

This is a public API removal (`pub fn from_legacy_sandbox_policy`), but it's an internal-protocol
crate within the codex-rs workspace, and the stack description confirms there are no remaining
non-test callers. Verification commands (`cargo test -p codex-protocol` and the targeted core test)
are well-scoped. No regression risk inside the migration; this is exactly the kind of cleanup that
should ride on top of the rest of the stack to prevent regression.

Merge as-is. The only nit would be that the deleted preset-equivalence test
(`permission_profile_presets_match_legacy_defaults`) is now no longer enforcing that
`PermissionProfile::read_only()` and `PermissionProfile::workspace_write()` agree with their legacy
projections, but #20398 is a removal-of-bridge PR and the equivalence is now enforced through the
cwd-aware constructor in the rest of the stack — losing this assertion is intentional, not a gap.
