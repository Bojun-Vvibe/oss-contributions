# openai/codex PR #20897 — Refactor app-server dispatch result flow

- Link: https://github.com/openai/codex/pull/20897
- Head SHA: `b7599fb44dbcdf33c287a569dcfe482eba1ccc55`
- Author: pakrym-oai
- Size: +1761 / −2526 (4 files, net −765)

## Files changed
- `codex-rs/app-server/src/codex_message_processor.rs` — `+1480 / −1929`
- `codex-rs/app-server/src/codex_message_processor/plugins.rs` — `+10 / −92`
- `codex-rs/app-server/src/codex_message_processor/thread_goal_handlers.rs` — `+92 / −212`
- `codex-rs/app-server/src/message_processor.rs` — `+179 / −293`

## Reasoning

This is a centralization refactor: the per-handler `to_connection_request_id(request_id)` shim and the per-arm `await` are replaced by a single `request_id` constructed up front and a unified `response: Result<Option<ClientResponsePayload>, JSONRPCErrorError>` collected from each match arm.

What I checked specifically:

1. `codex-rs/app-server/src/codex_message_processor.rs:1004` constructs the connection request id once before the match:
   ```
   let request_id = ConnectionRequestId {
       connection_id,
       request_id: request.id().clone(),
   };
   ```
   This is sound *only if* every match arm consumes `request_id` consistently — either by `.clone()` or by being the last arm to use it. The `ThreadStart` arm uses `request_id.clone()`, which means subsequent arms are also expected to clone. Skimming the diff this looks consistent, but with 1480 additions in one file the chance of one branch missing a clone (and silently shadowing the outer binding) is real. Worth a focused pass with `cargo expand` or just `rg "request_id\b" codex-rs/app-server/src/codex_message_processor.rs | wc -l` to confirm every arm is centralized the same way.
2. `codex-rs/app-server/src/codex_message_processor.rs:198-200` removes three protocol type imports (`CommandExecResizeParams`, `CommandExecTerminateParams`, `CommandExecWriteParams`, `ThreadRealtimeListVoicesParams`). PR claims branches with custom ordering remain "explicit"; need to verify those four request kinds are still actually handled — if the imports were unused because the arms now go through a uniform path, fine; if the arms were accidentally dropped along with the imports, that's a regression.
3. `codex-rs/app-server/src/message_processor.rs` `+179 / −293` is the cross-cutting change. PR description says "Replaced unreachable delegated request-family error arms with explicit `unreachable!` cases." `unreachable!` panics in production rather than returning a JSON-RPC error — that's a deliberate choice for invariants that *truly* cannot occur, but for a dispatch table it can convert a malformed-client bug into a server crash. Prefer `panic!("BUG: ...")` with context, or return a `JSONRPCErrorError::InternalError` with a logged message.

PR's own verification (`cargo check -p codex-app-server`, `cargo test -p codex-app-server thread_goal`, `just fix`) is appropriately scoped but only one test target was named — there's no signal that the full app-server test suite passes.

## Verdict

`merge-after-nits`

This is a high-leverage cleanup (−765 net LOC, fewer ad-hoc handler wrappers, easier-to-audit dispatch surface) and the design is right. Before landing:
- Confirm no `ClientRequest` arm was silently dropped along with the unused-import cleanup (e.g. `CommandExecResize`, `ThreadRealtimeListVoices`).
- Justify each new `unreachable!` in `message_processor.rs` or downgrade to a logged JSON-RPC internal error.
- Run the full `cargo test -p codex-app-server` (not just `thread_goal`) before merge.
