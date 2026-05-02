# openai/codex PR #20744 — fix(core) request_permissions tool flakey test

- **PR**: https://github.com/openai/codex/pull/20744
- **Head SHA**: `e7ea226be72d31fe274b06b25dfff118b5c9f695`
- **Size**: +41 / −41 in 1 file (`codex-rs/core/tests/suite/request_permissions_tool.rs`)
- **Verdict**: **merge-after-nits**

## What changed

Three coordinated test fixes:
1. `submit_turn` (line ~155 in the diff) now defaults `approvals_reviewer` to `Some(ApprovalsReviewer::User)` via `.or(Some(...))` when the caller passes `None`, instead of forwarding `None`.
2. `expect_request_permissions_event` (line ~180) tightens its `wait_for_event` predicate to match `RequestPermissions(request) if request.call_id == expected_call_id` directly, dropping the previous `RequestPermissions | TurnComplete` either-or and the panic branch on `TurnComplete`.
3. `apply_patch_after_request_permissions` (lines ~390–470): when `strict_auto_review` is true, the test now feeds the SSE harness **two** guardian responses (`{response_prefix}-guardian-1` and `-2`) instead of one, and the post-patch wait loop is restructured: a single `wait_for_event` now matches either `PatchApplyEnd { call_id == "apply-patch-call" }` or `ApplyPatchApprovalRequest { call_id == "apply-patch-call" }`, asserts `PatchApplyStatus::Completed` + `success`, then calls `wait_for_completion` unconditionally. The non-strict branch's redundant `ApplyPatchApprovalRequest | TurnComplete` polling block is deleted.

## Why this matters

The previous shape had two race surfaces. First, `expect_request_permissions_event` was asserting on event *ordering* by allowing `TurnComplete` to be matched and panicking on it — but a permission event for a different `call_id` could land first, get matched, and trigger an `assert_eq!` panic that looked like the wrong call_id rather than the real "we matched the wrong event" bug. The new predicate filters on `call_id` inside the matcher, so unrelated permission requests are simply skipped. Second, the strict-auto-review branch was racing the guardian SSE feed against the patch-apply pipeline: with only one guardian response queued, whether the test saw `PatchApplyEnd` first or got stuck waiting for a second guardian call was scheduling-dependent. Pre-queueing two guardian responses + waiting on a `call_id`-scoped patch event removes that race entirely. The `approvals_reviewer.or(Some(User))` default ensures every harness call ends up with a concrete reviewer rather than relying on production defaults.

## Specific call-outs

- The three changes are tightly coupled — splitting them would make the test transiently more flaky, not less. Keeping them in one PR is correct.
- Adding `use codex_protocol::protocol::PatchApplyStatus` (line 14 of the diff) is needed for the new `assert_eq!(end.status, PatchApplyStatus::Completed)`.
- Restructuring so `wait_for_completion(&test).await;` runs unconditionally (outside the `if strict_auto_review` block) is a real semantic improvement: previously the non-strict branch was inferring completion through a separate `TurnComplete` match that could swallow real failures.
- **Nit**: the `for review_index in 1..=2` loop hardcodes "two reviews" without a comment explaining *why* (presumably one for the apply_patch action + one for the guardian re-review of the result). Add a one-line comment so the next person doesn't think `=2` is a typo for `=1`.
- **Nit**: the panic branch `EventMsg::ApplyPatchApprovalRequest(approval) => panic!("unexpected apply_patch approval request after granted permissions: {:?}", approval.call_id)` is now reachable from both strict and non-strict modes. That's intentional and correct, but the message reads as if granted permissions should always skip approval — fine for this fixture, but worth a brief code comment that this asserts the post-permissions invariant.
- The deletion of the `else { let event = wait_for_event(…) … }` block (lines 469–485 in the diff) is the single largest behavioral change in the file; the new unified `wait_for_event` above it correctly subsumes both cases.

## Verdict rationale

Pure test-only change, well-targeted at a known flake source, and the restructuring reduces total lines while strengthening assertions. Both nits are documentation-only and shouldn't block. The risk is contained to this single test file, and the changes follow the project's existing event-matching idioms. CI signal will validate that the new predicates don't create *new* flakes in the strict path.
