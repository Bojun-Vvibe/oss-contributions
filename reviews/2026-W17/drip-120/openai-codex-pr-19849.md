# PR #19849 — Propagate runtime permission profiles

- **Repo**: openai/codex
- **PR**: #19849
- **Head SHA**: `5f6cf039`
- **Author**: evawong-oai
- **Size**: +116 / -21 across 2 files
- **URL**: https://github.com/openai/codex/pull/19849
- **Verdict**: **merge-as-is**

## Summary

Closes a hole in the runtime-sandbox-override path: when the user
flips the active sandbox policy mid-thread (e.g. via the TUI's
runtime override), the *next* turn-start was not actually telling
the agent the new permission profile. The TUI was sending the new
sandbox policy in the protocol envelope but leaving
`permission_profile` from the original config untouched, so the
remote side derived its profile from the now-stale config. Fix is
the right shape: at turn-start, derive the permission profile
*from the live runtime override* if one is set, and otherwise leave
the field None and let the receiver fall back to its own
config-derived default.

## Specific changes

- `codex-rs/tui/src/app/thread_routing.rs:530-617` — the destructure
  drops the `permission_profile` binding (now `_`), and the
  turn-start construction at `:614-617` replaces
  `permission_profile.clone()` with a call to the new helper
  `runtime_permission_profile_for_turn_start(self.runtime_sandbox_policy_override.as_ref(), sandbox_policy)`.
- `codex-rs/tui/src/app/thread_routing.rs:1485-1499` — the helper
  itself is delightfully terse: returns `None` if there is no
  runtime override (let the receiver fall back), returns `None` if
  the active policy is `ExternalSandbox` (those are derived from
  the external host's contract, not from a profile), and otherwise
  derives `PermissionProfile::from_legacy_sandbox_policy(sandbox_policy)`.
  This cleanly preserves the existing "no override → no override
  signal" contract.
- `codex-rs/tui/src/app/thread_routing.rs:1500-1530` — unit test
  `runtime_permission_profile_for_turn_start_only_when_sandbox_was_overridden`
  pins both arms: `None` override yields `None` profile;
  `Some(&policy)` override yields
  `Some(PermissionProfile::from_legacy_sandbox_policy(&policy))`.
  Exact pair-equality assertion on the legacy-derived value.
- `codex-rs/tui/src/app_server_session.rs:548,1107-1144` —
  `turn_start_permission_overrides` signature changes
  `sandbox_policy: SandboxPolicy` → `sandbox_policy:
  &SandboxPolicy` (avoids the move that previously forced clone-
  then-discard at the call site). The body is rewritten as a
  predicate-then-branch instead of a four-arm match, with the new
  predicate being "if remote *or* external-sandbox, send the legacy
  policy and no profile; otherwise embedded → send the profile and
  no policy".
- `codex-rs/tui/src/app_server_session.rs:1131-1146` —
  `permission_profile_override_from_config` now skips emitting a
  profile when the resolved legacy policy is `ExternalSandbox`,
  matching the same carve-out used by the runtime helper.
- `codex-rs/tui/src/app_server_session.rs:1529-1716` — three new
  test functions: embedded-with-runtime-profile,
  embedded-without-profile-yields-(None, None), and
  remote-keeps-legacy-policy. Together they fence off all three
  observable transitions across the matrix.

## Risks

Behavior change is only on the runtime-override path that was
previously broken. The `ExternalSandbox` carve-out is the right
call: external sandboxes (e.g. Seatbelt-supplied policy) own the
ground truth and the `from_legacy_sandbox_policy` derivation
doesn't have the right input bits to reconstruct it. Skipping
the profile field forces the receiver to use the policy field,
which is the existing pre-PR behavior for that variant.

Worth flagging that `PermissionProfile::from_legacy_sandbox_policy`
is itself a lossy projection (legacy policies don't carry every
field a profile does), but every caller in this PR already has a
legacy policy in hand and the receiver knows to treat the
derivation as a "best effort, fill in defaults" — not a behavior
this PR introduces.

## Verdict rationale

Surgical, well-tested, fixes a real "runtime sandbox override
silently doesn't take effect" UX bug. The four-arm-match → predicate
rewrite at `app_server_session.rs:1107-1124` is incidentally a
readability win. Merge as-is.
