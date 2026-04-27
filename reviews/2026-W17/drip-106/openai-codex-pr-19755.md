# openai/codex PR #19755 — Add Responses stream lifecycle diagnostics

- **Link**: https://github.com/openai/codex/pull/19755
- **Head SHA**: `1c2f5fea399c3f5c0bc17fde60d80096b753ff9e`
- **Author**: etraut-openai
- **Size**: +677 / -36 across 8+ files
- **Verdict**: `merge-after-nits`

## Files changed
- `codex-rs/codex-api/src/common.rs` — `ResponseEvent::Created` becomes a struct variant carrying `response_id: Option<String>`.
- `codex-rs/codex-api/src/endpoint/responses.rs` — adds `stream_request_with_lifecycle` / `stream_with_lifecycle` overloads taking `Option<ResponseStreamLifecycleOptions>`.
- `codex-rs/codex-api/src/endpoint/responses_websocket.rs` — same lifecycle threading for the WS transport; the run loop now wraps each terminal-error path with `finalize_lifecycle_error(&mut lifecycle, ResponseStreamTerminalState::*, error)` at three sites (lines ~597-621).
- `codex-rs/codex-api/src/sse/responses.rs` — analogous SSE-side wiring for the recorder.
- `codex-rs/codex-api/src/stream_lifecycle.rs` (new) — `ResponseStreamLifecycleRecorder`, `ResponseStreamLifecycleOptions`, `ResponseStreamTerminalState` enum (`StreamError`, `ClosedBeforeCompletion`, `IdleTimeout`, plus a successful-completion variant), and `finalize_lifecycle_error` helper that mutates the `ApiError` message to append a structured `Stream lifecycle: diagnostic: ...` suffix with `last_event=...` breadcrumb.
- `codex-rs/core/src/client.rs`, `codex-rs/core/src/session/turn.rs` — opt-in lifecycle threading from the call sites.
- `codex-rs/otel/src/events/session_telemetry.rs:1045-1048` — pattern-match update for the new struct variant `ResponseEvent::Created { .. }`.

## Analysis

The PR ships exactly what `## Why` promises: it makes "the stream died — but in which lifecycle phase?" a recoverable diagnostic rather than a transport-error blob. The `ResponseStreamTerminalState` enum is the right shape (created-but-silent, closed-before-completion, idle-timeout, stream-error, completed-with-mismatched-id), and the `finalize_lifecycle_error` pattern at `responses_websocket.rs` lines ~597-621 cleanly attributes each `Err` arm to one terminal state without changing the actual error type or retry budget. The integration test added near the diff tail asserts both the upstream provider message AND the new lifecycle-diagnostic suffix (`expected lifecycle diagnostic; got {err:?}` / `expected incomplete event diagnostic; got {err:?}` / `last_event=response.incomplete`) — that's the right level of coverage: it pins the rendered string contract that a downstream operator will grep for in logs.

The `ResponseEvent::Created` variant change from unit to `Created { response_id: Option<String> }` is the right move — currently the recorder can't distinguish "created event arrived but had no id" from "no created event at all" without it. The pattern-match update in `otel/src/events/session_telemetry.rs:1045-1048` correctly uses `Created { .. }` to ignore the field at the telemetry boundary.

The two-overload pattern (`stream_request` / `stream_request_with_lifecycle`, `stream` / `stream_with_lifecycle`) is fine for additive evolution but doubles the surface; a default of `None` on a single method would be cleaner if there are no external consumers of these names. Given the `pub` on both, there probably are.

## Nits to address before merge

1. **The `ResponseStreamTerminalState::IdleTimeout` arm in `responses_websocket.rs` line ~617** is currently bound to *any* `Err(err)` from the outer `select!` (the `Err(err) => { ... finalize_lifecycle_error(..., IdleTimeout, err) }`). That outer error path is from the idle-timeout `select!` arm specifically, but a future contributor reading the diff cold could easily think it covers any error. Add a `// SAFETY: this Err arm is only reached via the idle-timeout select branch` comment.
2. **`finalize_lifecycle_error` mutates the `ApiError` message in place by appending text**. Downstream code that `.contains()`-matches on error messages (there is some, especially in tests) will keep working, but anything that does an `==` comparison or depends on the exact prefix breaks. Sweep `tests/` and `core/` for `ApiError::Stream("stream closed before response.completed")` exact-match assertions and either update them or document the new contract in the recorder module.
3. **Telemetry coverage is asymmetric**: the SSE path emits the diagnostic but `session_telemetry.rs` only updates the variant pattern — does it record the lifecycle terminal state as a span attribute? If yes, add the attribute name to the PR description; if no, file a follow-up.
4. **Naming**: `ResponseStreamLifecycleRecorder::new(...)` is allocated unconditionally per-call when `lifecycle` is `Some`. For a high-QPS code path, confirm via a microbenchmark that the allocation isn't measurable, or gate with `#[cfg(feature = "stream-diagnostics")]`.

## What I learned

The pattern of "encode lifecycle phase as a typed terminal state, then mutate the error message at the boundary" is a nice middle ground between "introduce a new error variant per phase" (high blast radius) and "log it and lose it" (no operator value). The integration test that asserts both `last_event=response.incomplete` AND `Stream lifecycle: diagnostic:` as a **string contract** is the right way to lock log-grep behavior — once a runbook references that string, it's part of the public surface.
