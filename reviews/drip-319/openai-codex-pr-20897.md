# Review: openai/codex #20897 — Refactor app-server dispatch result flow

- PR: https://github.com/openai/codex/pull/20897
- Head SHA: `b7599fb44dbcdf33c287a569dcfe482eba1ccc55`
- Author: pakrym-oai
- Files touched (sample, large diff): `codex-rs/app-server/src/codex_message_processor.rs` (massive refactor of `dispatch_request`)

## Verdict: `merge-after-nits`

## Rationale

This is a structural refactor of `CodexMessageProcessor::dispatch_request` that converts ~50 per-arm `to_connection_request_id(request_id)` constructions into a single hoisted `request_id` (via `request.id().clone()`) and changes each arm from a `self.x(...).await;` statement into an expression that yields `Result<Option<ClientResponsePayload>, JSONRPCErrorError>`, presumably so a single response-dispatch tail at the bottom can serialize results centrally. The refactor is mechanically correct on the arms I read (`ThreadStart`, `ThreadResume`, `ThreadFork`, `ThreadArchive`, etc.) — every closure now uses `request_id.clone()` which is necessary because the binding is reused across many arms. Three concerns: (1) cloning `ConnectionRequestId` for every arm is fine if the inner `RequestId` is cheap, but for arms that previously *moved* `request_id` into the call this is now an extra clone — worth `git blame`'ing whether `RequestId` is `Arc`-backed (then no concern) or a `String` (then ~50 small allocations per dispatch on the hot path). (2) The diff removes three imports (`CommandExecResizeParams`, `CommandExecTerminateParams`, `CommandExecWriteParams`, `ThreadRealtimeListVoicesParams`) which suggests those arms changed to use them through a different path — confirm they aren't accidentally orphaned. (3) Changing every method's call shape means callers' return types must now be `Result<Option<ClientResponsePayload>, JSONRPCErrorError>`; if any of the ~50 helper methods still send their own response and return `()`, the refactor will double-respond. Need to see the full diff (>1000 lines) and the helper method signatures to be confident; nits are about verifying invariants the diff alone can't show.