# openai/codex PR #20110 — tests: add no-wait helper for apply patch event tests

- Repo: `openai/codex`
- PR: https://github.com/openai/codex/pull/20110
- Head SHA: `9e4bfa31a4a10094ec2fc9724402e5a78b87d0e0`
- State: OPEN, +104/-54 across 2 files

## What it does

Repairs the regression #20040 introduced in the `apply_patch_cli` test suite. #20040 migrated those tests off manual `Op::UserTurn` construction onto the profile-backed `harness.submit(...)` helpers, but `harness.submit` blocks until `TurnComplete` and drains the event stream that the same tests then assert against (`TurnDiff`, `PatchApplyUpdated`, …). Net effect: tests passed locally, broke on Linux/Windows Bazel CI with empty-event-stream races.

This PR keeps the profile-backed approach but adds a parallel `submit_turn_without_wait_*` family alongside the existing `submit_turn_with_*` family — same signatures, but they fire and return without consuming `TurnComplete`, so the test body retains ownership of the event stream.

## Specific reads

- `core/tests/common/test_codex.rs` new helpers: `submit_turn_without_wait_with_permission_profile`, `submit_turn_without_wait_with_approval_and_permission_profile`, and the private `submit_turn_without_wait_with_permission_profile_context`. Each shadows its `submit_turn_with_*` sibling with the *same* parameter list — that symmetry is load-bearing because the `apply_patch_cli` test edits become a 1:1 method-name substitution with no parameter reshuffling.
- The shared bottom of the helper chain is the new `submit_turn_op_with_context(...)` private fn (~line 800). The old `submit_turn_with_permission_profile_context` is rewritten to delegate to it and *then* do the `wait_for_event_match(TurnStarted) → wait_for_event_with_timeout(TurnComplete, SUBMIT_TURN_COMPLETE_TIMEOUT)` block; the new `submit_turn_without_wait_*` path returns immediately after `submit_turn_op_with_context`. Single source of truth for the op-construction half — good factoring.
- The `wait_for_event_with_timeout(... TurnComplete ...)` after the wait-for-`TurnStarted` swallows the timeout (no `?` or `expect`). That preserves prior behavior but means the *waiting* helper variant silently passes through a missed `TurnComplete`. Not introduced by this PR, but the refactor was the moment to either propagate that or comment it.
- The diff is +104/-54 across 2 files, but only `core/tests/common/test_codex.rs` carries the helper definitions; the second file is presumably one apply_patch_cli test file edited to call the new no-wait helpers. The helper-only change is test infrastructure with zero production impact.

## Risk

- Adds 4 nearly-identical method bodies in the harness API surface. If a future change adds another orthogonal axis (say `service_tier`), there are now 2× the helpers to keep in sync.
- The `_without_wait_` naming reads ambiguous at call sites — "wait for what?" — without a docstring. A one-line comment "does not wait for TurnComplete; caller owns the event stream and must drain it" on `submit_turn_without_wait_with_permission_profile` would prevent the next migration from reintroducing the same race.

## Verdict

`merge-after-nits` — correct fix for the right reason: don't widen `submit_turn`'s contract to be conditionally-blocking, add a parallel non-blocking family. The two nits are docstring and the long-standing swallowed-timeout, both small. Test-only change, zero blast radius outside the harness.
