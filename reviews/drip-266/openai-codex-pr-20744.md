# openai/codex #20744 — fix(core) request_permissions tool flakey test

- **Repo:** openai/codex
- **PR:** #20744
- **URL:** https://github.com/openai/codex/pull/20744
- **Head SHA:** `e7ea226be72d31fe274b06b25dfff118b5c9f695`
- **Files touched:** `codex-rs/core/tests/suite/request_permissions_tool.rs` (+41 -41)
- **Verdict:** merge-as-is

## Summary

Pure test stability fix. Two distinct flake sources are addressed:

1. The matcher in `expect_request_permissions_event` was
   `RequestPermissions(_) | TurnComplete(_)`, then asserted `call_id`
   inside the match arm. Because event ordering between
   `RequestPermissions` and a closely-following `TurnComplete` is not
   strictly serialized in the channel, the test could match an *earlier*
   permission request (different `call_id`) and then panic on the
   assertion. The new pattern `RequestPermissions(request) if
   request.call_id == expected_call_id` filters at the predicate level so
   only the matching event is dequeued.
2. `apply_patch_after_request_permissions` in the strict-auto-review
   path was racing the guardian SSE response: the test sent one guardian
   reply but the system sometimes performed two guardian round-trips
   under load. The fix queues two guardian SSE responses (loop
   `1..=2` at lines 391-405) and, more importantly, restructures the
   wait so it consumes either `PatchApplyEnd` (success path) or
   `ApplyPatchApprovalRequest` (regression-detect path, panics with
   diagnostic) before falling through to `wait_for_completion`.

## Specific notes

- **`request_permissions_tool.rs:155`** —
  `approvals_reviewer: approvals_reviewer.or(Some(ApprovalsReviewer::User))`
  defaults the reviewer to `User` if the caller didn't specify, which
  pins the review path deterministically. Good.
- **Lines 440-461** — the new `wait_for_event` block is the ordering
  fix; both branches assert state, and `wait_for_completion(&test).await`
  is now called *unconditionally* (line 460) rather than only in the
  strict branch. Symmetric and clearer.
- **Loop at lines 391-405** — a slight wart is that in the
  non-strict path you still queue two guardian responses (loop runs
  unconditionally inside `if strict_auto_review`). Wait — re-reading,
  it's correctly inside the `if strict_auto_review` block. Disregard.

## Why merge-as-is

Test-only change, narrowly scoped, the predicate-level filter is the
correct pattern for these flaky-by-design event-stream tests, and the
diagnostic panic on `ApplyPatchApprovalRequest` preserves regression
visibility. No production code touched.
