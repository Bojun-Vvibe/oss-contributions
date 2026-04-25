# codex #19589 — Fix request_permissions tool flake in core tests

- URL: https://github.com/openai/codex/pull/19589
- Head SHA: `7e53fc270ed25c6275a3533657b9c0fbbe4e761b`
- Verdict: **merge-as-is**

## What it does

Two `request_permissions_tool` integration tests
(`approved_folder_write_request_permissions_unblocks_later_exec_without_seatbelt`
and `apply_patch_after_request_permissions`) were doing a one-shot lookup
on the wiremock-recorded function-call output and panicking when the
output hadn't been written yet. The fix adds a `Notify`-based wait helper
to the shared `ResponseMock` and switches the two test sites to await it.

## Diff notes

- New plumbing in `core/tests/common/responses.rs`:
  ```
  pub struct ResponseMock {
      requests: Arc<Mutex<Vec<ResponsesRequest>>>,
      requests_updated: Arc<Notify>,
  }
  ```
  with `wait_for_function_call_output_text(call_id)` doing the canonical
  "subscribe before checking" pattern inside a `tokio::time::timeout(10s)`:
  ```
  let requests_updated = self.requests_updated.notified();
  if let Some(output) = self.function_call_output_text(call_id) { return output; }
  requests_updated.await;
  ```
  That's the correct ordering — calling `.notified()` before the check
  prevents the lost-wakeup race where a notify fires between the check
  and the await.
- The matcher is updated to call `notify_waiters()` after pushing the
  request body. Multiple waiters get woken up, which is fine because each
  waiter re-checks its own predicate before returning.
- Test sites switch from `function_call_output_text(...).unwrap_or_else(panic)`
  to `wait_for_function_call_output_text(...).await`. The `json!({"output": ...})`
  wrap is preserved verbatim.

## Risk surface

- 10-second timeout panics with `"timed out waiting for {call_id} output"`
  if something genuinely deadlocks. That's the right failure mode — flake
  becomes a deterministic timeout with a useful message rather than a
  silent never-finishing test.
- `notify_waiters` (vs `notify_one`) is correct here: every waiter has its
  own `call_id` predicate, so they can't starve each other.

## Why this verdict

This is a textbook fix for a tokio test-flake — replace polling/one-shot
lookups with a `Notify` subscribe-then-check loop. No behavioral change
to production code, only to the shared test mock and two consumers.
Lands clean.
