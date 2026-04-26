# PR #19589 — Fix request_permissions tool flake in core tests

- **Repo**: openai/codex
- **PR**: #19589
- **Head SHA**: `7e53fc270ed25c6275a3533657b9c0fbbe4e761b`
- **Author**: dylan-hurd-oai
- **Size**: +29 / -10 across 2 files
- **Verdict**: **merge-as-is**

## Summary

Two integration tests in `request_permissions_tool.rs` were
polling `responses.function_call_output_text(call_id)` once
and panicking with `expect("expected exec-call output")` if
the mock hadn't yet recorded that request. Under load the
mock recorder and the assertion racing produced sporadic CI
failures. This PR replaces the one-shot poll with a notified
wait that loops on a `tokio::sync::Notify` until the value is
present, with a 10-second hard timeout.

## Specific changes

- `codex-rs/core/tests/common/responses.rs:42` — `ResponseMock`
  gains `requests_updated: Arc<Notify>` field.
- `responses.rs:84-95` — new
  `wait_for_function_call_output_text(call_id)` async helper:
  ```rust
  tokio::time::timeout(Duration::from_secs(10), async {
      loop {
          let requests_updated = self.requests_updated.notified();
          if let Some(output) = self.function_call_output_text(call_id) {
              return output;
          }
          requests_updated.await;
      }
  })
  .await
  .unwrap_or_else(|_| panic!("timed out waiting for {call_id} output"))
  ```
  The pattern of *registering the notified-future before
  checking the condition* (line `let requests_updated =
  self.requests_updated.notified();` *before* the
  `function_call_output_text` check) is the textbook
  race-free wait pattern with `Notify`. Without that
  ordering, a notification fired between the check and the
  await would be lost. The author got this exactly right.
- `responses.rs:598-611` — `Match::matches` now wraps the
  push in a fresh inner block (`{ self.requests.lock()...
  push(...); }`) so the lock guard drops before
  `requests_updated.notify_waiters()` fires. Avoids waking a
  waiter while it's still contending on the same mutex.
  Subtle and correct.
- `tests/suite/request_permissions_tool.rs:300-305` and
  `:463-468` — both call sites switch from the panicky
  `function_call_output_text(...).unwrap_or_else(|| panic!(...))`
  to `.wait_for_function_call_output_text(...).await`. The
  surrounding `json!({ "output": exec_output })` wrap is
  preserved — just no longer fused with the unwrap.

## Risks

Very low. Two notes:

1. **10-second timeout** is hardcoded. If CI gets slower this
   may start re-flaking. Worth a `const FLAKE_GUARD_TIMEOUT`
   so it's tunable in one place.
2. Other tests in this suite using
   `function_call_output_text` directly are still subject to
   the same race. This PR fixes the two known offenders;
   would be worth grepping for other one-shot pollers to
   convert pre-emptively.
3. The `notify_waiters` approach wakes *all* current waiters
   — if multiple tests share a `ResponseMock`, that's fine
   because each waiter re-checks its own `call_id`. But it
   does mean spurious wakeups; the loop handles that
   correctly by re-arming `notified()` and re-checking.

## Verdict

`merge-as-is` — clean test-flake fix using the canonical
`Notify` wait pattern, with the lock-drop-before-notify
detail handled correctly. Two cleanups (timeout constant,
sibling poll sites) are follow-up material, not blockers.

## What I learned

Two patterns worth internalizing from this diff:

1. **Register the wait before the check.** With
   `Notify::notified()`, you create the future *first*, then
   check the condition, then `await`. Reversing the order
   creates a window where a notification is dropped on the
   floor.
2. **Drop the mutex guard before notifying.** Notifying
   waiters who immediately try to re-acquire the same lock
   is a self-inflicted contention storm. The new inner block
   `{ self.requests.lock().unwrap().push(...); }` is the
   minimum-viable scope-narrowing for that.
