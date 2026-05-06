# block/goose#8870 — fix(cli): emit cumulative token usage in stream-json/json complete events

- **Head SHA**: `961bbe0c27698e6c82a5c732076d58d9362f0f7c`
- **Stats**: +28 / -5, 1 file (`crates/goose-cli/src/session/mod.rs`)

## Summary

The CLI's JSON output mode and stream-JSON `Complete` event were emitting only `total_tokens`, even though `SessionMetadata` separately tracks `accumulated_input_tokens` / `accumulated_output_tokens` / `accumulated_total_tokens` (the cumulative-across-turns counters that are the right thing to bill / display) plus their per-last-turn equivalents. Downstream consumers parsing `--output stream-json` couldn't distinguish input vs output spend, and couldn't see the cumulative figure when a session spanned multiple turns. This PR adds optional `input_tokens` and `output_tokens` fields to both the final `JsonMetadata` (non-streaming) and the `StreamEvent::Complete` (streaming) payloads, and routes all three through the `accumulated_*.or(per_turn_*)` precedence chain.

## Specific citations

- `crates/goose-cli/src/session/mod.rs:65-72`: `JsonMetadata` gains `input_tokens: Option<i32>` and `output_tokens: Option<i32>`, both `#[serde(skip_serializing_if = "Option::is_none")]`. The skip-if-none is the right call — preserves backward compat (existing consumers parsing `JsonMetadata` won't see new keys when the data isn't available) and keeps JSON minimal.
- `:88-95`: `StreamEvent::Complete` mirror — same two new fields, same skip-if-none. Symmetric with the non-streaming variant, good.
- `:1273-1283`: in the non-streaming success branch, all three token fields use `session.accumulated_<x>_tokens.or(session.<x>_tokens)`. The `or()` chain is the load-bearing semantic: prefer cumulative if present, fall back to last-turn. The error branch correctly emits `None` for all three.
- `:1294-1313`: stream-JSON branch refactors the previous single-field lookup into a tuple destructure. The `match session { Some(s) => (...), None => (None, None, None) }` pattern is cleaner than three chained `.and_then(...)` and surfaces the "session-fetch failed" case as all-None which `skip_serializing_if` then drops from the wire.

## Verdict

**merge-after-nits**

## Rationale

Right-sized fix. The cumulative-vs-per-turn precedence (`accumulated_*.or(per_turn_*)`) is the correct billing/display semantic — most downstream tooling wants "how many tokens did this session cost," not "how many in the last turn." Symmetric treatment of streaming and non-streaming paths is good hygiene. Three nits before merge: (1) **no test coverage** added — for a behavioral wire-format change that downstream automation parses, even one fixture-based test asserting the `input_tokens`/`output_tokens` keys appear in `serde_json::to_string` output when accumulated counters are populated, and are absent when both are `None`, would lock the contract. (2) The PR description doesn't enumerate which downstream consumers (CLI JSON mode users, ACP transports, telemetry pipelines) currently parse the stream-JSON Complete event — confirming none rely on `total_tokens` being the *only* token field would close a backward-compat risk. (3) Schema documentation: if there's a published JSON schema for `StreamEvent::Complete` (likely under `crates/goose-sdk/`), it should be updated in the same PR — two-step "code lands, schema lags" is the usual source of SDK drift. None block merge for the fix itself, but the missing test is the largest open question.
