# openai/codex#20430 — turn-context: stop writing legacy sandbox policy

- PR: https://github.com/openai/codex/pull/20430
- Head SHA: `7a367c3db7...`
- Author: bolinfest
- Files: 8 changed, ~+15 / −40 (test fixtures + protocol shape)

## Context

This PR is one rung in the multi-PR bolinfest stack migrating turn-context payloads from the legacy `SandboxPolicy` enum to the new `PermissionProfile` representation (drip-207 covered #20422 and #20406, both of which prepared callers to consume `PermissionProfile`). Up through this point, `TurnContext::to_turn_context_item` was double-writing — emitting `permission_profile` *and* still computing a legacy projection into `sandbox_policy` so older callers could keep reading. This PR flips writers off: producers stop populating `sandbox_policy` and `file_system_sandbox_policy`, the protocol field becomes `Option<SandboxPolicy>` with `skip_serializing_if = "Option::is_none"`, and the readback path falls back to deriving the profile from the legacy field only when present (preserves rollout-replay compat for files written before the migration).

## Design

Three coordinated changes:

1. **`session/turn_context.rs:303-322`** — deletes `non_legacy_file_system_sandbox_policy()` (the equivalence-with-legacy-projection helper that was the only thing keeping `file_system_sandbox_policy` populated), and `to_turn_context_item` now emits `sandbox_policy: None, file_system_sandbox_policy: None` unconditionally. `permission_profile` becomes the only source of truth on the wire.
2. **`protocol/protocol.rs:2856-2860, 2885-2900`** — `TurnContextItem.sandbox_policy: SandboxPolicy` becomes `Option<SandboxPolicy>` with `#[serde(default, skip_serializing_if = "Option::is_none")]`. `permission_profile()` reader now panics with a precise message (`"turn context item must contain permission_profile or legacy sandbox_policy"`) when *both* are missing, and otherwise reconstructs from whichever is present.
3. **Test sweep across `core/src/session/{rollout_reconstruction_tests,tests}.rs`, `core/tests/suite/resume_warning.rs`, `rollout/src/recorder_tests.rs`, `state/src/extract.rs`, `tui/src/{app/tests,lib}.rs`, `core/src/context_manager/history_tests.rs`** — every test fixture either wraps the existing legacy value in `Some(...)` (back-compat read paths that need to test that branch) or sets `sandbox_policy: None` (write-path tests that should match the new producer behavior).

## What's interesting

The asymmetry between writers and readers is the right tactic for protocol migrations like this. Writers stop emitting *now* (so that all newly-rolled-out code starts producing the new shape immediately). Readers continue to accept the legacy field via the fallback in `permission_profile()` indefinitely, so old rollout files keep replaying. The panic instead of a silent default in the dual-missing case is the right call — that combination only happens if a malformed payload comes from outside the codebase.

The test rename at `session/tests.rs:5949` (`turn_context_item_stores_split_file_system_sandbox_policy_when_different` → `turn_context_item_stores_split_file_system_policy_in_permission_profile`) and the assertion change to `assert_eq!(item.sandbox_policy, None)` and `assert_eq!(item.file_system_sandbox_policy, None)` is the right semantic: the per-cwd file-system policy is now stored *inside* the `PermissionProfile`, not as a sibling top-level field. The companion rename at `session/tests.rs:6075` is consistent.

## Risks / nits

- `extract.rs:303-310` keeps a reader-side test that constructs a `TurnContextItem` with `sandbox_policy: None` and `permission_profile: Some(PermissionProfile::Disabled)` — and the next two extract tests (line 305, 320) construct fixtures with `sandbox_policy: Some(...), permission_profile: None` to exercise the legacy-fallback path. Good — both branches of the new `permission_profile()` are covered.
- The `panic!` in `permission_profile()` will fire on any externally-constructed `TurnContextItem` that omits both fields. A `Result` would be cleaner, but the function signature is `-> PermissionProfile` (not `Result`) and changing it would ripple through every caller. The panic message names both alternatives, which is actionable for anyone debugging a malformed rollout. Acceptable for now.
- This PR is part of a 25+ PR stack (#20398 → #20406 → #20420 → #20422 → #20430 → ...). Reviewers approving in isolation should sanity-check that no readers in the stack downstream of this one re-introduce a `.sandbox_policy.unwrap()` pattern. A `git grep -E "sandbox_policy(\.[a-z]+\(\))" codex-rs` on the head commit would catch that.

## Verdict

**merge-as-is** — clean stack-step PR, test sweep matches the writer-flip cleanly, and the back-compat reader path is preserved with explicit panic on the impossible-via-correct-producer case. Nothing to nit; the stack discipline here is exactly what migrations like this need.
