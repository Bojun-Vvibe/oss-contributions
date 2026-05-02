# openai/codex PR #20686 — test: make async cancellation test deterministic

- URL: https://github.com/openai/codex/pull/20686
- Head SHA: `5ba42268e892f4a10531a3eb2725ff20d2dea620`
- Author: bolinfest
- Verdict: **merge-as-is**

## Summary

Replaces a flaky timing-based test for `or_cancel` in `codex-rs/async-utils/src/lib.rs` with a deterministic one that uses `std::future::pending()` as the inner future. The previous version raced a 100ms `sleep` against a `cancel_handle` task: under CI load the sleep could complete first, and the assertion `Err(CancelErr::Cancelled)` would fail. The new version uses an inner future that *never resolves*, so the only path out of `or_cancel` is the cancellation token, making the assertion trivially deterministic.

## Line-level observations

- `codex-rs/async-utils/src/lib.rs` line 37: `use std::future::pending;` is the new import — minimal and correct.
- `codex-rs/async-utils/src/lib.rs` lines 62–63: the inner future changes from `async { sleep(100ms).await; 7 }` to `pending::<i32>()`. `pending::<i32>()` returns `Pending` forever, so `or_cancel` can only complete via the cancellation branch. The explicit turbofish `::<i32>` is needed because the future has no other type information; would not compile without it. Good.
- The `cancel_handle` task is unchanged — it still spawns and calls `token_clone.cancel()`. The race that previously existed (sleep vs cancel) is now eliminated because there is no sleep on the inner-future side.
- `sleep` and `Duration` imports in the surrounding `mod tests` block are retained because other tests in the module still use them. Diff is appropriately minimal.
- Behavior under failure: if `or_cancel` regresses and *doesn't* observe the cancellation, the test will hang forever (because the inner future is `pending`). Tokio's test runtime defaults will kill the test eventually, but a `tokio::time::timeout` wrapper around the assertion would convert a regression from "hang then panic" to "fail fast with a clear message." Minor; not worth blocking.
- The test continues to use `pretty_assertions::assert_eq` and `task::spawn` — no surprises.

## Suggestions

1. Optional: wrap the assertion in `tokio::time::timeout(Duration::from_secs(5), ...)` so a regression where `or_cancel` ignores the token surfaces as a clean failure rather than a runtime-killed hang. The fix in this PR is the right one regardless.
2. Optional: rename the test to make the determinism guarantee explicit (e.g. `or_cancel_returns_cancelled_when_inner_never_resolves`) — current name (presumably the same as before) understates what the test now proves.

Otherwise this is a clean, minimal flake fix. Ship it.
