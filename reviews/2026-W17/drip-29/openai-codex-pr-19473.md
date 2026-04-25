# openai/codex#19473 — Add turn start timestamp to turn metadata

- PR: https://github.com/openai/codex/pull/19473
- Author: mchen-oai
- +209 / -12
- Head SHA: `a7cd879e9742b56a1864995605345f29dd9c4989`

## Summary

Propagates a new `turn_started_at_unix_ms` field into the
`x-codex-turn-metadata` header so MCP servers can measure their own latency
relative to turn start. The field is sourced from
`TurnTimingState::mark_turn_started`, which is upgraded from second-precision
to millisecond-precision and now returns the start instant so the caller can
fan it out to `TurnMetadataState`. Reserved-key handling is hardened: a
client-supplied `turn_started_at_unix_ms` in `responsesapi_client_metadata` is
now ignored (server value wins), matching how `session_id`, `thread_source`,
and `turn_id` are already protected.

## Specific findings

- `codex-rs/core/src/turn_timing.rs:55` (SHA
  `a7cd879e9742b56a1864995605345f29dd9c4989`) — `mark_turn_started` now
  returns `i64` (unix ms) and computes `started_at_unix_secs` from
  `started_at_unix_ms / 1000`. Good: single source of truth, no skew between
  the seconds field and the new ms field. The `now_unix_timestamp_secs()`
  wrapper is preserved as `now_unix_timestamp_ms() / 1000`, so existing
  callers don't observe a behavior change.
- `codex-rs/core/src/turn_metadata.rs:78` — `merge_responsesapi_client_metadata`
  is renamed to `merge_turn_metadata` and grows a `turn_started_at_unix_ms:
  Option<i64>` parameter. The early-return `if turn_started_at_unix_ms.is_none()
  && responsesapi_client_metadata.is_none() { return None; }` correctly preserves
  the prior behavior where no client metadata meant "don't even parse the base
  header." The inserted `metadata.insert(TURN_STARTED_AT_UNIX_MS_KEY, ...)`
  uses `insert` (not `entry().or_insert`), which is the right call —
  server-set values must win over any client-supplied ms field.
- `codex-rs/core/src/turn_metadata.rs:115` — the explicit `if key ==
  TURN_STARTED_AT_UNIX_MS_KEY { continue; }` guard inside the client-metadata
  merge loop is the security-relevant line: without it, a client could shadow
  the server's ms value via `responsesapi_client_metadata`. The new
  `turn_metadata_state_ignores_client_turn_started_at_unix_ms_before_start`
  test at `turn_metadata_tests.rs:142` covers the regression cleanly.
- `codex-rs/core/src/tasks/mod.rs:303` — the call site captures the returned
  ms and immediately forwards it to `turn_context.turn_metadata_state`. Order
  matters: this happens *after* `mark_turn_started` writes the timing state,
  so any subsequent `current_header_value()` consumer sees both the timing
  state and the metadata field consistently.
- `codex-rs/core/src/turn_metadata_tests.rs:93` — new
  `assert!(json.get("turn_started_at_unix_ms").is_none());` assertion in the
  pre-existing platform-sandbox test pins down the "absent unless explicitly
  set" contract. Good defensive test.

## Verdict

`merge-as-is`

## Rationale

Tight, focused change with the right test coverage: a positive case
(timestamp present after `set_turn_started_at_unix_ms`), a negative case
(client cannot inject the field), and a merged-with-other-metadata case
(server wins). The seconds→ms refactor in `turn_timing.rs` is backward
compatible because `started_at_unix_secs` is still computed and surfaced
unchanged. No locking or ordering concerns — `turn_started_at_unix_ms` is a
plain `Arc<RwLock<Option<i64>>>` written once per turn before any MCP tool
call header is built.
