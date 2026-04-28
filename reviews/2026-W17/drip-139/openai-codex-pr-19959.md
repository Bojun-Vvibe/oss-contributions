# openai/codex #19959 — Fix log db batch flush flake

- PR: https://github.com/openai/codex/pull/19959
- Author: dylan-hurd-oai (Dylan Hurd)
- Head SHA: `aabb04b726fc`
- Base: `main` · State: OPEN
- Diff: 2+/1- across 1 file (`codex-rs/state/src/log_db.rs`)

## Verdict: merge-as-is

## Rationale

- **Right diagnosis of a real `tokio::time::interval` footgun.** `tokio::time::interval` defaults to `MissedTickBehavior::Burst` and *fires immediately on first call to `.tick()`* — the first tick happens at t=0, not t=interval. So at `log_db.rs:367-369`, the inserter loop's `select!` arm on `ticker.tick()` would fire instantly on the first iteration, regardless of whether any entries had been buffered. Adding `ticker.tick().await` at `:370` (with the comment "Consume the immediate startup tick so entries flush after the interval") consumes that spurious first tick before the loop body runs. This is the canonical tokio-interval fix and matches how `actix-rt` and `tracing-subscriber` solve the same problem.
- **The test deletion is the load-bearing half.** `tests/.../log_db.rs:648` removes `tokio::time::sleep(std::time::Duration::from_millis(10)).await` from `configured_batch_size_flushes_without_explicit_flush`. That sleep was a *symptom-mask*: it gave the spurious-startup-tick enough wall-clock to fire and flush an empty buffer before the test started writing entries, hiding the race between "first tick at t=0" and "test starts writing at t≈0+ε". With the production fix, the sleep is no longer needed — and *keeping* it would mask future regressions of the same class. Test plan in the PR body explicitly notes "fails before implementation patch with tightened test" and "3 consecutive passes" — the right signal-strength evidence for a flake fix.
- **Two-line change with no API surface impact.** The `run_inserter` function signature is unchanged; the new `ticker.tick().await` is internal scheduling discipline that callers can't observe. Risk of regression in adjacent code paths is essentially zero — the only behavioral change is "first flush happens at t=interval instead of t=0," which is the documented contract that everyone expected anyway.

## Nits / follow-ups

- The inline comment "Consume the immediate startup tick so entries flush after the interval" is good but could cite `MissedTickBehavior::Burst` as the underlying cause for grep-back when someone hits the same shape elsewhere. A future maintainer seeing only the comment might not realize it's a tokio-API quirk vs. an arbitrary delay.
- Worth grepping the rest of `codex-rs/` for other `tokio::time::interval(...)` constructions without a leading `.tick().await` consumption — same bug class likely exists in any other long-running periodic flusher (`metrics_exporter`, `heartbeat`, log rotation if any). One-line fix per call site.
- Test name `configured_batch_size_flushes_without_explicit_flush` is doing real work (it's testing that the size-trigger fires without needing a manual `flush()` call), but the new contract being pinned is also "and doesn't fire spuriously at startup" — worth a sibling test `configured_batch_size_does_not_flush_empty_at_startup` that pushes zero entries and asserts the flush count stays at zero through one interval boundary, to pin the inverse half of the contract.

## What I learned

`tokio::time::interval` is a widely-mined footgun whose default behavior — fire immediately, then again every `period` — surprises ~everyone the first time. The structural tell of this bug is "intermittent flake in a test that exercises a periodic flusher, masked by a leading `sleep(small_duration)` in the test setup." That sleep is the *evidence* the test author hit the same race and patched it locally without realizing the production code was the cause. The right fix pattern is always: (1) consume one `.tick()` at the top of the producer loop, (2) delete every test-side sleep that was masking the race, (3) note the cause in a comment so the next person doesn't reintroduce a sleep "to make it less flaky." This PR hits all three.
