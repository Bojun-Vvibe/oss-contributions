# Review: openai/codex#20815

- **PR:** openai/codex#20815
- **Head SHA:** `b8da0c99f672163689327f1d551e739b01121bb3`
- **Title:** Improve /side fork and replay performance
- **Author:** etraut-openai

## Files touched (high-level)

- `codex-rs/app-server-protocol/schema/json/{ClientRequest,codex_app_server_protocol,codex_app_server_protocol.v2,v2/ThreadForkParams}.json` — add new `ThreadForkHistoryLimit` schema entry and a `historyLimit?: ThreadForkHistoryLimit` field on `ThreadForkParams`.
- `codex-rs/app-server-protocol/schema/typescript/v2/ThreadForkHistoryLimit.ts` (new) + `index.ts` re-export.
- `codex-rs/app-server-protocol/src/protocol/v2.rs` — Rust definition of `ThreadForkHistoryLimit { max_turns, since_last_compaction }` plus the new optional field on `ThreadForkParams`.
- `codex-rs/app-server/README.md` — docs for the new `historyLimit` option on `thread/fork`.
- `codex-rs/app-server/src/codex_message_processor.rs` — destructure `history_limit`, validate `max_turns > 0`, and feed it into `fork_thread_from_history` via a new `Self::thread_fork_snapshot(history_limit.as_ref())` helper instead of the hardcoded `ForkSnapshot::Interrupted`.

## Specific observations

- `protocol/v2.rs` (~ line 3782): `ThreadForkHistoryLimit` uses `#[experimental("thread/fork.historyLimit.maxTurns")]` and `#[experimental("thread/fork.historyLimit.sinceLastCompaction")]` per-field, plus `ts(optional = nullable)` on the optional `max_turns`. That matches existing experimental field patterns in this file (e.g. `persistExtendedHistory`).
- `protocol/v2.rs` ~3860: new `pub history_limit: Option<ThreadForkHistoryLimit>` on `ThreadForkParams` is gated by `#[experimental("thread/fork.historyLimit")]`. Good — clients without the experimental shape stay unaffected.
- `codex_message_processor.rs:4960-4980`: the validation
  ```rust
  if history_limit.as_ref().and_then(|limit| limit.max_turns).is_some_and(|max_turns| max_turns == 0) {
      return Err(invalid_request("`historyLimit.maxTurns` must be greater than zero"));
  }
  ```
  is correct, but it accepts `max_turns: None` and `since_last_compaction: false` simultaneously, i.e. an empty `historyLimit: {}`. That's effectively a no-op — consider rejecting it explicitly so clients don't silently get the same behavior as omitting `historyLimit` entirely (or document that it's a no-op).
- `codex_message_processor.rs:5054`: replacing `ForkSnapshot::Interrupted` with `Self::thread_fork_snapshot(history_limit.as_ref())` is the load-bearing change. The diff doesn't show the body of `thread_fork_snapshot`, but it must (a) preserve the `Interrupted` marker behavior when `history_limit` is `None`, and (b) honor both `max_turns` and `since_last_compaction` when set. Worth verifying that the snapshot returned actually constrains the `InitialHistory::Resumed(ResumedHistory { ... })` payload that follows — otherwise the limit only affects the snapshot marker, not the copied turns.
- `since_last_compaction: bool` with `#[serde(default, skip_serializing_if = "std::ops::Not::not")]` — fine, but note the experimental field will silently default to `false` on older clients; semantically clear, just worth a sentence in the README hunk (the README hunk does cover this).
- Schema regeneration looks consistent across `codex_app_server_protocol.schemas.json`, `*.v2.schemas.json`, `v2/ThreadForkParams.json`, and the TypeScript `ThreadForkHistoryLimit.ts` + `index.ts` re-export. Mechanical and correct.

## Verdict

**merge-after-nits**

## Reasoning

This is a clean, narrowly-scoped extension of `thread/fork` that adds two well-defined history-bounding knobs (`maxTurns`, `sinceLastCompaction`) behind the experimental flag, with matching schema regeneration and README updates. The Rust-side validation correctly rejects `max_turns == 0`. Two small follow-ups before merge: (1) decide whether `historyLimit: {}` (both fields unset) should be rejected as ambiguous or explicitly documented as a no-op, and (2) confirm in the body of `thread_fork_snapshot` (not visible in the head-200 of the diff) that the limit actually trims the copied `ResumedHistory` rather than only updating the `ForkSnapshot` marker. Both are 5-minute verifications.
