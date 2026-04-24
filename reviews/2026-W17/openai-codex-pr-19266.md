---
pr_number: 19266
repo: openai/codex
head_sha: 2a73ee1de05959278c13372be66403469ca6a55f
verdict: merge-after-nits
date: 2026-04-24
---

# openai/codex#19266 — non-local thread-store regression harness

**What changed.** Three things, tied together:

1. `codex-rs/app-server/src/codex_message_processor.rs` switches `configured_thread_store` from string-based `experimental_thread_store_endpoint.as_deref()` to a typed `ThreadStoreConfig` enum match: `Local`, `Remote { endpoint }`, and (debug-only) `InMemory { id }`. The `#[cfg(debug_assertions)]` gate on `InMemoryThreadStore::for_id(id)` (line 34) keeps the test-only store out of release binaries.
2. New test module `app-server/tests/suite/v2/remote_thread_store.rs` (254 lines, debug-only) wires the in-memory store through the same `ConfigBuilder` → `in_process::start` path the real server uses.
3. The actual assertion is the interesting one: after `thread/start` + `turn/start`, the temp `codex_home` must contain **no** rollout files and **no** sqlite state files. Stops a regression class where local persistence quietly materializes alongside a configured non-local store.

**Why it matters.** "Configured remote, but local files keep showing up" is exactly the kind of silent dual-write bug that's invisible until a customer asks "why is my disk filling up?"

**Concerns.**
1. **The harness only catches *artifact* leaks, not *probe* leaks** (the comment at line 71 acknowledges this). A read-only probe of the local store that returns no rows leaves no file but is still wrong behavior. Worth a follow-up that wraps the local store in a "must not be called" decorator and asserts zero method calls.
2. **`InMemoryThreadStore::for_id(store_id)` keyed by random UUID** (line 124) means the test instance is held alive only by the static for-id map until the test exits. If two tests in the same process collide on store_id it would be a real bug — `Uuid::new_v4()` makes that vanishingly unlikely but document the intent at the for-id call site.
3. **`assert_eq!(thread.path, None)`** (line 161) is the right invariant for non-local stores, but a single-line assert with no message will print `Some("/tmp/...")` on failure with no context. Add an `, "non-local store must not return a local rollout path"` message.
4. **Timeout of 10 s for the turn-completed loop** (line 106) is fine for CI noise, but the inner `loop { client.next_event().await }` consumes every event. If the server emits an unexpected `TurnFailed`, the loop ignores it and waits for `TurnCompleted` until timeout. Match `TurnFailed` and bail with the failure reason.
5. **`ThreadStoreConfig` enum** is the real shipping change; the test is the safety net for it. Worth landing the enum even on its own — string-matching `experimental_thread_store_endpoint` was always ugly.

Useful regression net for a real footgun. Land after the assertion-message and `TurnFailed` polish.
