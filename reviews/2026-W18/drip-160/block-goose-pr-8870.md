# block/goose PR #8870 — fix(cli): emit cumulative token usage in stream-json/json complete event

- Repo: `block/goose`
- PR: https://github.com/block/goose/pull/8870
- Head SHA: `961bbe0c27698e6c82a5c732076d58d9362f0f7c`
- State: OPEN, +28/-5 across 1 file (`crates/goose-cli/src/session/mod.rs`)

## What it does

Fixes an under-reporting bug in the `--output-format json` and `--output-format stream-json` CLI emitters: the "completion" event was reading only `session.total_tokens` (the *current turn* count), not the *cumulative session* counts. For multi-turn sessions this consistently under-reported usage by N-1 turns. Fix prefers `session.accumulated_total_tokens.or(session.total_tokens)` (cumulative if present, fall back to current-turn for older session formats) and adds the previously-missing `input_tokens` and `output_tokens` fields to both the `JsonMetadata` struct and the `StreamEvent::Complete` enum variant.

## Specific reads

- `crates/goose-cli/src/session/mod.rs:1276-1280` — the JSON-mode metadata fix:
  ```rust
  Ok(session) => JsonMetadata {
      total_tokens: session.accumulated_total_tokens.or(session.total_tokens),
      input_tokens: session.accumulated_input_tokens.or(session.input_tokens),
      output_tokens: session.accumulated_output_tokens.or(session.output_tokens),
      status: "completed".to_string(),
  },
  ```
  The `Option::or` chain is the correct shape: cumulative wins, fall back to per-turn for sessions persisted under an older format that didn't track accumulated counters. Note the **fallback semantic shift** for older sessions: a multi-turn session that was started under a pre-accumulated-counter version will now report only its *last* turn's counts under `total_tokens`, not zero — same as today, no regression.

- `crates/goose-cli/src/session/mod.rs:1294-1313` — the stream-json mode parallel fix:
  ```rust
  let session = self.agent.config.session_manager.get_session(&self.session_id, false).await.ok();
  let (total_tokens, input_tokens, output_tokens) = match session {
      Some(s) => (
          s.accumulated_total_tokens.or(s.total_tokens),
          s.accumulated_input_tokens.or(s.input_tokens),
          s.accumulated_output_tokens.or(s.output_tokens),
      ),
      None => (None, None, None),
  };
  emit_stream_event(&StreamEvent::Complete { total_tokens, input_tokens, output_tokens });
  ```
  Same `Option::or` chain duplicated here — could be hoisted into a small helper `fn token_counts(s: &Session) -> (Option<i32>, Option<i32>, Option<i32>)` to keep the two emitters in lockstep on future field additions, but the duplication is short enough to be readable.

- `crates/goose-cli/src/session/mod.rs:67-70` and `:91-94` — schema additions:
  ```rust
  #[serde(skip_serializing_if = "Option::is_none")]
  input_tokens: Option<i32>,
  #[serde(skip_serializing_if = "Option::is_none")]
  output_tokens: Option<i32>,
  ```
  `skip_serializing_if = "Option::is_none"` on the new fields is the right call for backward compatibility: existing JSON consumers parsing the `Complete` event won't see new fields appear on sessions where the underlying counters are absent, so a strict-schema consumer won't suddenly fail. **But**: when `total_tokens` is present and the new `input_tokens`/`output_tokens` are absent, downstream cost-modeling code that subtracts (output = total - input) silently produces nonsense. A note in the release log or a structured comment near the field declarations would help.

- The error-branch parallel at line 1281-1285:
  ```rust
  Err(_) => JsonMetadata {
      total_tokens: None,
      input_tokens: None,
      output_tokens: None,
      status: "completed".to_string(),
  }
  ```
  Status remains `"completed"` even when the session metadata fetch failed — that's a load-bearing invariant the diff preserves (callers expect the CLI to claim "completed" on graceful exit even if the post-completion read fails). Reasonable but worth confirming intent in the PR description; the diff just preserves prior behavior.

## Verdict: `merge-as-is`

## Rationale

This is a small, surgical fix to a real under-reporting bug that consistently produced misleading numbers in scripted CLI consumers. The `accumulated_X.or(X)` chain is the right primitive (cumulative-preferred, single-turn-fallback for legacy session shapes), the `skip_serializing_if = "Option::is_none"` decoration on the new fields preserves wire-format backward compatibility for strict consumers, and the diff keeps the JSON and stream-json emitters in lockstep. No tests in the diff but the surface is small (one file, three structurally-identical sites) and behavior is verifiable by manual `goose run --output-format stream-json` against a multi-turn session. Could optionally hoist a `(total, input, output)` triple helper to keep the two emitters DRY, and could optionally add a release-notes line warning cost-modelers not to assume `output = total - input` when only `total` is present, but neither is blocking.
