---
pr: 19626
repo: openai/codex
sha: 32b698812e63a0f0dcdde0dc5e7f8e0e4f1deb1c
verdict: merge-after-nits
date: 2026-04-26
---

# openai/codex #19626 — Preserve assistant phase for replayed messages

- **Author**: friel-openai
- **Head SHA**: 32b698812e63a0f0dcdde0dc5e7f8e0e4f1deb1c
- **Link**: https://github.com/openai/codex/pull/19626
- **Size**: ~80 diff lines across `protocol/src/models.rs`, `protocol/src/protocol.rs`, `core/src/codex_thread.rs`, `core/src/goals.rs`, `core/src/session/{handlers,mod,tests}.rs`, `core/src/tasks/user_shell.rs`.

## Scope

`ResponseInputItem::Message` (the inbound side of the conversation protocol) only carried `role` + `content`, while `ResponseItem::Message` (the outbound/persisted side) carried `phase: Option<MessagePhase>`. The asymmetry meant that any time an assistant `MessagePhase::Commentary` item was round-tripped through `ResponseInputItem::from(...) → ResponseItem::from(...)` (which happens on replay, on inter-agent injection, on user-shell output persistence, and on every queued/pending input flush), the phase was silently dropped to `None` and the assistant's commentary collapsed into a normal turn. This PR adds the missing field and threads it through.

## Specific findings

- `codex-rs/protocol/src/models.rs:608-615` — `ResponseInputItem::Message` gains `#[serde(default, skip_serializing_if = "Option::is_none")] #[ts(optional)] phase: Option<MessagePhase>`. Wire-compatible: `default` means existing serialized inputs without the field still deserialize, `skip_serializing_if = Option::is_none` means producers that don't set it still emit byte-identical JSON. TypeScript bindings get `phase?: MessagePhase` via `#[ts(optional)]`. Good.
- `codex-rs/protocol/src/models.rs:1046-1058` — the `From<ResponseInputItem> for ResponseItem` conversion now propagates `phase` instead of hard-coding `None`. This is the actual bugfix.
- `codex-rs/protocol/src/models.rs:1598-1622` — new `response_input_message_conversion_preserves_phase` test pins exactly this (`Commentary` survives the conversion). Good.
- `codex-rs/protocol/src/protocol.rs:855-862` — `InterAgentCommunication::into_response_input_item()` now sets `phase: Some(MessagePhase::Commentary)` so cross-agent messages render as commentary. Reasonable default; this is a semantic decision and worth flagging in the PR body in case other agents wanted plain phase here.
- `codex-rs/core/src/codex_thread.rs:441-449` — `pending_message_input_item` now destructures `phase` from `ResponseItem::Message` and propagates it. Good — this is the direct round-trip path.
- `codex-rs/core/src/tasks/user_shell.rs:341-349` — same pattern in `persist_user_shell_output`. Good.
- All the `From<Vec<UserInput>>`, goals.rs steering items, session handlers injection sites add `phase: None` explicitly. These are user-originated or system-developer-originated messages where `None` is correct, and the explicit `None` makes the choice intentional rather than incidental. Fine, though somewhat noisy across `goals.rs:1310`, `session/handlers.rs:1268`, `session/mod.rs:1688`, `:1785`, `tests.rs:6112,6359,6362,6365,6404,6431,6451`.
- The destructure in `core/src/session/tests.rs:6657` uses `Message { role, content, .. }` (with rest-pattern) — fine, but inconsistent with the `Message { role, content, phase }` destructures elsewhere. Either pick one style or the rest-pattern is fine since `phase` is unused there.

## Risk

Low. Wire-compatible field add (default + skip_if_none), no migration. The semantic risk is sites that should preserve `phase` but still hard-code `None`; I scanned all 8 call-sites in the diff and they're correct (user-input, dev-injection, and steering should all default to `None`; the two round-trip sites correctly propagate). One test added at the conversion seam.

## Verdict

**merge-after-nits** — would benefit from a behavioral test showing an end-to-end round-trip (replay → inject → persist) preserves `Commentary` phase, not just the unit conversion. The current test catches the obvious regression but not the subtler "did we miss a call site" class.
