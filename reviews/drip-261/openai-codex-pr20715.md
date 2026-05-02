# openai/codex PR #20715 — Make realtime sideband startup async

- **Head SHA:** `c5fd3a8cc79418f3bdf458dac78195f36cbf320b`
- **Files:** 2 (`core/src/realtime_conversation.rs`, `core/tests/suite/realtime_conversation.rs`)
- **LOC:** +187 / −88

## Observations

- `realtime_conversation.rs:199-205` — introducing `RealtimeInputChannels` to bundle `(user_text_rx, handoff_output_rx, audio_rx)` is a tidy refactor that makes the sideband-vs-direct branching readable. The struct is plain-data, no traits — good.
- `realtime_conversation.rs:292-355` — the real semantic change: when an SDP is provided, the `client.connect_webrtc_sideband(...)` call is moved *out of* `RealtimeConversationManager::start()` and *into* a spawned task `spawn_webrtc_sideband_input_task`. This means `start()` returns to the caller immediately after `model_client.create_realtime_call_with_headers(...)` succeeds, and the sideband WebSocket handshake races with subsequent `Op::RealtimeConversationText` submits. The new test at `realtime_conversation.rs:513-595` explicitly exercises this by submitting text *before* the handshake completes and asserting the queued text arrives at request-index 1 — good coverage of the new contract.
- `realtime_conversation.rs:215-218` — `ConversationState` lost its `writer: RealtimeWebsocketWriter` field. Anything that previously reached into `state.writer` directly (e.g. for ad-hoc sends or shutdown) now has to go through the input task's channels. Worth grepping the rest of the codebase to confirm no caller still expects `state.writer`.
- `realtime_conversation.rs:1054-1072` — `connect_webrtc_sideband` failure path: if `realtime_active` is still set, the error is mapped, logged via `warn!`, and pushed onto `events_tx` as `RealtimeEvent::Error(...)`. If `realtime_active` was already cleared (e.g., user closed the conversation while connecting), the failure is silently dropped. That matches the desired "don't surface late errors after teardown" semantic but please confirm no caller depends on a guaranteed terminal event.
- The `accept_delay: Some(Duration::from_millis(250))` change in the test (`realtime_conversation.rs:476`) is what unmasks the race — previously the immediate accept hid the synchronous handshake. Good test design.

## Verdict: `merge-after-nits`

- Grep for residual `state.writer` consumers; the field removal is the riskiest line.
- Consider whether silent drop on `!realtime_active` post-failure is observable (debug log at minimum).
