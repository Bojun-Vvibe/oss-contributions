---
pr: 19630
repo: openai/codex
sha: 8f32415619da8e25485e86a696fe8201db0397bb
verdict: merge-as-is
date: 2026-04-26
---

# openai/codex #19630 — Avoid persisting ShutdownComplete after thread shutdown

- **Author**: etraut-openai
- **Head SHA**: 8f32415619da8e25485e86a696fe8201db0397bb
- **Link**: https://github.com/openai/codex/pull/19630
- **Fixes**: #19475 (likely regression from #18882)
- **Size**: ~62 diff lines across `codex-rs/core/src/session/{handlers.rs,tests.rs}`.

## Scope

`codex exec` was emitting `ERROR failed to record rollout items: thread <id> not found` on stderr **after** a successful run completed. Root cause: shutdown closes the `LiveThread` writer in `ThreadStore` *before* the terminal `ShutdownComplete` event is emitted, but the terminal event was still routed through `send_event_raw`, which in turn tries to append the event to the (already-removed) live writer. Result: a benign-but-noisy stderr line that wrappers treating stderr-as-failure misinterpret as a failed exec and retry.

## Specific findings

- `codex-rs/core/src/session/handlers.rs:978-986` — the fix swaps `sess.send_event_raw(event).await` for a two-step:
  1. `sess.services.rollout_thread_trace.record_protocol_event(&event.msg)` — records to the in-memory protocol-event trace, which does not depend on the live thread writer
  2. `sess.deliver_event_raw(event).await` — delivers to subscribers without going through the rollout/thread-store append path
  This is exactly the right shape: `ShutdownComplete` is the one event whose existence a `ThreadStore.shutdown_thread` predates, so it must use the non-store-touching delivery path.
- `codex-rs/core/src/session/tests.rs:4336-4373` — `shutdown_complete_does_not_append_to_thread_store_after_shutdown` is gated `#[cfg(debug_assertions)]` and asserts on `InMemoryThreadStoreCalls { create_thread: 1, shutdown_thread: 1, ..Default::default() }`. The `..Default::default()` rest-pattern is the key assertion — it pins that *zero* `append`/`record_event`/etc. calls land after `shutdown_thread`. Good — this is the regression-prevention test.
- The `#[cfg(debug_assertions)]` gate is unusual for an assertion of this kind. Likely because `InMemoryThreadStoreCalls`/`calls()` is debug-only instrumentation. Worth a one-line comment to make that explicit.
- The PR body honestly flags `#18882` as the likely-but-unbisected source. That's the right level of caution.

## Risk

Very low. The change is two lines of substance plus one regression test. `deliver_event_raw` already handles the non-store path correctly for other code sites (this just routes the terminal event through it). Wire format unchanged — the same `ShutdownComplete` event still ships to subscribers.

## Verdict

**merge-as-is** — minimal, targeted, with a regression test that pins the exact invariant the bug violated (no thread-store calls after `shutdown_thread`). The optional comment on `#[cfg(debug_assertions)]` is documentation polish, not a blocker.
