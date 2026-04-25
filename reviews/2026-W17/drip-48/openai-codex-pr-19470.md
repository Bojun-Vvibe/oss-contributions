# openai/codex#19470 — Add turn start timestamp to turn metadata

- **URL**: https://github.com/openai/codex/pull/19470
- **Author**: mchen-oai
- **Head SHA**: `30ee6637ac4fbac320f9d8818a434b04d2df8212`
- **Verdict**: `merge-after-nits`

## Summary

Plumbs a `turn_started_at_unix_ms` integer (millis since epoch) into the
per-turn metadata header so MCP servers and the responses API receive a
consistent turn start timestamp. The value is captured at task dispatch
in `tasks/mod.rs` from `mark_turn_started` and stored on
`TurnMetadataState` via a new `Arc<RwLock<Option<i64>>>` field; the
`merge_responsesapi_client_metadata` helper is renamed to
`merge_turn_metadata` and now folds both the timestamp and existing
client metadata into the header JSON.

## Reviewable points

- `core/src/tasks/mod.rs` lines around 301–310 now bind the return value
  of `mark_turn_started(started_at).await` to
  `turn_started_at_unix_ms` and immediately call
  `turn_context.turn_metadata_state.set_turn_started_at_unix_ms(...)`.
  Order is fine — the metadata write happens before any tool dispatch
  could read the header — but a comment justifying why the write must
  precede `enrichment_task` startup would help future maintainers.
- `core/src/turn_metadata.rs`: the new `merge_turn_metadata` correctly
  short-circuits to `None` only when **both** inputs are absent
  (lines around 78–82). The `if key == TURN_STARTED_AT_UNIX_MS_KEY {
  continue; }` guard at the responsesapi merge step is the right call —
  it keeps `merge_turn_metadata` authoritative if a caller accidentally
  puts the same key in `responsesapi_client_metadata`. Worth a one-line
  test asserting the precedence.
- The new test `mcp_tool_call_request_meta_includes_turn_started_at_unix_ms`
  in `mcp_tool_call_tests.rs:686` only checks the custom-server path.
  The codex-apps path also routes through `build_mcp_tool_call_request_meta`
  (per the existing `codex_apps_tool_call_request_meta_includes_turn_metadata_and_codex_apps_meta`
  test) and should get the same assertion to prevent regressions on the
  metadata merge.
- `i64` millis is fine for ~292M years; no overflow concern. Using a
  signed type matches the existing `set_turn_started_at_unix_ms`
  contract on `TurnTimingState` so the value passes through unchanged.

## Rationale

This is a small, well-scoped enrichment; the renamed helper now has a
clearer responsibility and the lock ordering is symmetric with the
existing `responsesapi_client_metadata` field. Nits are non-blocking:
add a codex-apps test covering the new key and a comment near the
`set_turn_started_at_unix_ms` call documenting the ordering invariant.
Otherwise mergeable.
