---
pr_number: 8870
repo: block/goose
head_sha: 961bbe0c27698e6c82a5c732076d58d9362f0f7c
verdict: merge-after-nits
date: 2026-04-28
---

# block/goose#8870 — fix(cli): emit cumulative token usage in stream-json/json complete event

**What changed.** Single-file +28/−5 in `crates/goose-cli/src/session/mod.rs`. Three correlated changes in `JsonMetadata` and `StreamEvent::Complete` shape + emission:

1. **New optional fields** at `:67-70` (`JsonMetadata`) and `:91-94` (`StreamEvent::Complete`): `input_tokens: Option<i32>` and `output_tokens: Option<i32>`, both gated by `#[serde(skip_serializing_if = "Option::is_none")]` so old consumers parsing the JSON shape don't see new keys when the values are unset.
2. **Field source flip for `total_tokens`** at `:1276` and `:1303-1306`: `session.total_tokens` → `session.accumulated_total_tokens.or(session.total_tokens)`. The field name and position stay the same in the wire format — only its source changes from "last turn's context size" to "running sum across all turns" with last-turn fallback for legacy/empty sessions.
3. **Symmetric pattern** for the new `input_tokens`/`output_tokens` at `:1278-1280` and `:1307-1309`, both following the same `accumulated_X.or(X)` precedence.

**Why it matters.** PR body's analysis is correct and load-bearing: `Session::total_tokens` is overwritten on every LLM round-trip and reset to summary-output count after compaction (per `update_session_metrics` in `crates/goose/src/agents/reply_parts.rs`). Eval harnesses and billing dashboards parsing `--output-format stream-json` reasonably interpret `total_tokens` as cumulative, so they currently *under-report by an order of magnitude on multi-turn runs*. The PR cites a concrete reproduction: thousands of streamed chunks reporting `total_tokens=17099` (final-turn context only).

**Concerns.**
1. **Backwards-compat claim is correct.** `total_tokens` keeps its name, position, and `Option<i32>` type. Existing parsers see a *larger correct number* where they previously saw an under-count. This is the right kind of breaking-but-not-shape change: anyone consuming `total_tokens` for billing/usage was already broken; the fix corrects them. New `input_tokens`/`output_tokens` fields are opt-in via `skip_serializing_if`.
2. **`accumulated_total_tokens.or(session.total_tokens)` precedence is right** — `accumulated_X` is the cumulative truth when present (multi-turn session that emitted at least one usage event), falls back to `total_tokens` for empty/legacy sessions where `accumulated_*` is `None`. Verify that `accumulated_total_tokens` is `Some(0)` rather than `None` for a session that ran zero LLM calls — if it's `Some(0)`, the fallback is dead code; if it's `None` for the empty case, the `.or()` correctly catches it.
3. **No unit test added** despite a 28-line behavioral change. PR checklist box is unchecked: "`cargo test -p goose-cli` : please run in CI". For an output-shape change consumed by external tools, a one-cell test pinning the JSON wire format would be cheap insurance against future drift. Even a snapshot-style assertion that `serde_json::to_string(&JsonMetadata { total_tokens: Some(100), input_tokens: Some(60), output_tokens: Some(40), status: "completed" })` produces the documented shape.
4. **Both error-path branches** at `:1281-1283` and `:1310` correctly default to `None` for all three fields. No silent `Some(0)` confusion for the failure case.
5. **`JsonOutput` (the `--output-format json` case)** at `:1284` and the `--output-format stream-json` case at `:1306-1310` are now *almost* symmetric in their token-extraction code — both compute the same `(s.accumulated_X.or(s.X), ...)` triple. A small refactor extracting `fn token_summary(session: &Session) -> (Option<i32>, Option<i32>, Option<i32>)` would prevent the two sites from drifting on the next field addition (e.g. `cache_read_tokens`).
6. **`println!("{}", serde_json::to_string_pretty(&json_output)?)`** at the json-mode path uses pretty-printed output, while `emit_stream_event` (presumably) uses compact. The test surface needs to cover both, since pretty-printed output spans multiple lines and some consumer parsers may be line-buffered.
7. **`StreamEvent` derives** (`Serialize, Deserialize`?) — confirm the deserialize path round-trips the new optional fields cleanly for CI consumers that re-parse the event stream into typed shapes.

Real bug, correct fix, backwards-compatible wire format. Ship after adding even a single round-trip test (concern 3).
