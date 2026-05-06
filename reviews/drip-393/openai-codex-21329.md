# Review: openai/codex#21329

- **PR:** [feat: include thread ID in MCP turn metadata](https://github.com/openai/codex/pull/21329)
- **Head SHA:** `790c150fd02b1be30df2ef20942604e281ad3e7c`
- **Merged:** 2026-05-06T09:36:16Z
- **Verdict:** `merge-as-is`

## Summary

Splits the historically-conflated `session_id`/`thread_id` identities in MCP turn metadata. `session_id` continues to be inherited by descendant (review/sub-agent) threads so consumers can correlate at session granularity; `thread_id` is now also emitted so consumers can pin work to the concrete thread. Both flow through the new `Session::session_id()` / `Session::thread_id()` accessors.

## Specific notes

- `core/src/session/session.rs:328-336` — new accessors are minimal and well-documented. `thread_id()` returns `self.conversation_id` (the local thread identity) and `session_id()` delegates to `self.services.agent_control.session_id()` (the inherited shared id). Naming finally matches the underlying semantics; previously the code used `conversation_id` for the thread-local id which was misleading.
- `core/src/session/turn_context.rs:441-446` — `make_turn_context()` now takes both `thread_id` and `session_id` explicitly; all call sites updated. Nothing left passing one in for both.
- `core/src/session/review.rs:103-108` — review thread spawn now correctly seeds `TurnMetadataState` with `(session_id, thread_id)` separately. Before, the review thread's metadata bag was being seeded with `conversation_id` for both fields, which would have leaked the parent's id as the "thread" id once this change landed elsewhere — this PR fixes that in lockstep.
- `core/src/turn_metadata.rs:69-73` — `thread_id: Option<String>` added to `TurnMetadataBag` with `skip_serializing_if = "Option::is_none"`. Serialization is forward-compatible: old consumers see one extra field, new consumers can rely on it being present for new turns.
- `core/src/session/tests.rs:4060-4097` — `resumed_root_session_uses_thread_id_as_session_id` and `resumed_subagent_session_keeps_inherited_session_id` updated to assert *both* `thread_id` and `session_id` independently. This is the right shape: prior assertion only covered the inherited id and would have missed regressions in the local thread id.

## Rationale

This is a clean disambiguation of two identities that were conflated in the type system. Tests cover the two scenarios that matter (root resume, sub-agent resume), the wire format change is additive and skip-serializes when absent, and every call site of `make_turn_context` is updated to pass both ids explicitly so the type system enforces the new contract. Approve as-is.
