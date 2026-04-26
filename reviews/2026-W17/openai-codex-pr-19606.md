---
pr: 19606
repo: openai/codex
sha: 8eff69b41c924549282c10b0f35637a5cad96dbb
verdict: needs-discussion
date: 2026-04-26
---

# openai/codex #19606 — permissions: make runtime config profile-backed

- **Author**: bolinfest
- **Head SHA**: 8eff69b41c924549282c10b0f35637a5cad96dbb
- **Size**: ~4836 diff lines across 62 files. Supersedes #19391.
- **Builds on**: #19604 (test-only base), #19231 (introduced `PermissionProfile`).

## Scope

`PermissionProfile` (Managed/Disabled/External + filesystem rules) became canonical in #19231 but couldn't represent everything `SandboxPolicy` did and vice-versa, so config-loading was forcing every profile through the strict legacy `SandboxPolicy` bridge — which *rejected* valid runtime profiles like direct write-roots. This PR makes `PermissionProfile` first-class on both `Permissions` and `SessionConfiguration`, keeps `sandbox_policy` as a **legacy projection** (compatibility view) rather than the source of truth, and uses `compatibility_sandbox_policy_for_permission_profile(...)` for legacy consumers.

## Specific findings

- `codex-rs/protocol/src/models.rs` (per grep) — `PermissionProfile` is now constructed via `from_runtime_permissions_with_enforcement(enforcement, ...)` rather than `from_legacy_sandbox_policy(&SandboxPolicy::DangerFullAccess)`. The new constructor takes the enforcement mode (`Managed`/`Disabled`/`External`) explicitly instead of inferring it from the legacy policy shape. This is the right direction — inferring enforcement from `SandboxPolicy::DangerFullAccess` was always lossy. Tests that previously did `from_legacy_sandbox_policy(&SandboxPolicy::DangerFullAccess)` are migrated to `PermissionProfile::Disabled` (e.g. diff line 175 `let full_access_profile = codex_protocol::models::PermissionProfile::Disabled;`).
- The dual-path projection: at every call site that needs a `SandboxPolicy` for a legacy consumer (sandbox tags, exec request construction, prompt permission text), the diff inserts `let sandbox_policy = compatibility_sandbox_policy_for_permission_profile(&effective_permission_profile, ...)`. This means `SandboxPolicy` is computed *from* the profile on demand, not stored alongside it — eliminating the drift risk.
- `Permissions.permission_profile` and `SessionConfiguration.permission_profile` are added as the canonical fields. Setters keep "PermissionProfile, split filesystem/network policies, and legacy SandboxPolicy projections synchronized" — see PR body. Worth verifying there is exactly *one* mutating setter per struct so the synchronization invariant is enforced by construction; if there are multiple setter paths, some can desync.
- The PR body's claim "preserves configured deny-read entries and `glob_scan_max_depth` when command/session profiles are narrowed" is a concrete behavioral promise. I did not find a regression test pinning this in the diff slice I sampled — given this is exactly the class of bug that would be silent (denied-read still gets read, depth limit silently widened), a targeted test asserting both fields survive a narrowing transform is essential before merge.
- `PermissionProfile::read_only()` and `::workspace_write()` presets are added "to match legacy defaults". Net positive — collapses legacy literal `SandboxPolicy::ReadOnly` / `SandboxPolicy::WorkspaceWrite` constructions that were scattered across config-loading. Risk: any divergence between the new presets and the legacy constants is silent. A unit test asserting `PermissionProfile::read_only().to_legacy_sandbox_policy() == SandboxPolicy::ReadOnly` (and likewise for workspace_write) would pin the parity.

## Risk

**High structural, low behavioral if migration is correct.** This is the canonical "wrong abstraction at the bottom" refactor — replacing the source-of-truth field across 62 files. The class of bugs to fear is asymmetric: a missed call site that still consults `sandbox_policy` directly instead of going through the profile will silently use a *stale* policy projection after a profile mutation. Catching this requires (a) confirming `sandbox_policy` is never re-mutated independently of the profile (only re-projected), and (b) round-trip tests at every config-loading entry point.

## Verdict

**needs-discussion** — the refactor direction is correct and overdue, but at 4836 lines / 62 files it carries real review burden. Asks before merge: (1) is there a single canonical setter for `permission_profile` that updates the projection in lockstep, with all other writes routed through it; (2) regression tests for the deny-read + glob_scan_max_depth preservation claim; (3) parity tests for `read_only()`/`workspace_write()` presets against the legacy constants; (4) a grep-audit confirming no remaining direct mutations of `Permissions.sandbox_policy` outside the projection path. If those four are addressed, this becomes merge-after-nits.
