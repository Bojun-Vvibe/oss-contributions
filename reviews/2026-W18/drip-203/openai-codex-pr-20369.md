# openai/codex #20369 — guardian: configure review session permissions directly

- **Author:** bolinfest
- **SHA:** `cfeaa5a`
- **State:** OPEN
- **Size:** +5 / -6 across 1 file
  (`codex-rs/core/src/guardian/review_session.rs`)
- **Verdict:** `merge-as-is`

## Summary

Removes the legacy `SandboxPolicy → PermissionProfile` bridge from the
guardian-review-session config builder. Before:
`build_guardian_review_session_config` at
`guardian/review_session.rs:898-904` constructed a
`SandboxPolicy::new_read_only_policy()`, fed it through
`PermissionProfile::from_legacy_sandbox_policy(&sandbox_policy)` for the
`Constrained::allow_only(...)` slot, and *also* called
`set_legacy_sandbox_policy(sandbox_policy, guardian_config.cwd.as_path())` to
mutate the live config — round-tripping the same intent through two distinct
abstractions. After: the canonical `PermissionProfile::read_only()` is built
once at `:899`, used directly for both the `Constrained::allow_only(...)` slot
at `:900-901` and the in-place `set_permission_profile(permission_profile)`
mutation at `:903`.

## Reasoning

This is a textbook small-surface cleanup with three correctness arguments:

1. **The behavior is observably identical.** `PermissionProfile::read_only()`
   is the canonical constructor; `PermissionProfile::from_legacy_sandbox_policy(&SandboxPolicy::new_read_only_policy())`
   was the documented round-trip equivalent. The only previous reason to go
   through the legacy bridge was that `set_legacy_sandbox_policy` needed the
   raw `SandboxPolicy` value — but that setter was always doing
   `PermissionProfile::from_legacy_sandbox_policy(...)` *internally* before
   storing, so the new direct path eliminates one redundant conversion per
   review-session bootstrap without changing the stored value.

2. **The error message at `:907` correctly tracks the API rename.** Old:
   "guardian review session could not set sandbox policy: {err}". New:
   "guardian review session could not set permission profile: {err}". The
   message now names the actual operation that can fail (`set_permission_profile`)
   rather than the deprecated wrapper, which will save someone five minutes
   in the next on-call.

3. **The PR body explicitly preserves `Op::UserTurn.sandbox_policy`
   compatibility.** This is the right call — that field is part of the
   external request protocol and removing it would be a breaking change.
   The legacy bridge is only being removed from internal-construction paths
   where the entire data flow stays inside the crate, which is exactly where
   the cleanup has zero blast radius.

No tests are added but none are warranted — the change is a refactor of the
construction path with no new branches or exit conditions, and the existing
guardian-review-session integration tests in
`codex-rs/core/tests/guardian/` continue to exercise the read-only-permissions
behavior end-to-end. If those go green in CI the change is provably
behavior-preserving.

Ship it. The next dependency-removal step (whatever still calls
`PermissionProfile::from_legacy_sandbox_policy` from inside the crate) is a
clean follow-up — `rg "from_legacy_sandbox_policy" codex-rs/core/src/` will
list the remaining sites.
