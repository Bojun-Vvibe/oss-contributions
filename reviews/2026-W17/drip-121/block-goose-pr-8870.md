# block/goose #8870 — fix(cli): emit cumulative token usage in stream-json/json complete event

- URL: https://github.com/block/goose/pull/8870
- Head SHA: `2f60a434a3bf4586bb153b289095d91b2ea6cc0d`
- Diff: +36/-5 in `crates/goose-cli/src/session/mod.rs`.

## Context / problem

The `complete` event in `goose run --output-format stream-json` (and `metadata.total_tokens` in `--output-format json`) was emitting `Session::total_tokens`, which is the **last turn's context size** — overwritten on every LLM round-trip, and reset to the *summary output count* after compaction (see `update_session_metrics` in `crates/goose/src/agents/reply_parts.rs`). Downstream eval harnesses and billing dashboards interpret `total_tokens` as cumulative session cost, so they under-report by an order of magnitude on multi-turn runs. The PR author hits a real number: a session that streamed thousands of chunks reported `total_tokens=17099` (just the final-turn context) instead of the actual cumulative usage.

## What the fix does

`session/mod.rs:67-78` extends `JsonMetadata`:

```rust
struct JsonMetadata {
    /// Cumulative tokens across the whole session ...
    /// Mirrors `Session::accumulated_total_tokens`, **not** `Session::total_tokens`...
    total_tokens: Option<i32>,
    #[serde(skip_serializing_if = "Option::is_none")]
    input_tokens: Option<i32>,
    #[serde(skip_serializing_if = "Option::is_none")]
    output_tokens: Option<i32>,
    status: String,
}
```

`session/mod.rs:91-105` extends `StreamEvent::Complete` with the same two new optional fields. Both new fields use `skip_serializing_if = "Option::is_none"` so they're omitted from the JSON payload when unset — strict-schema downstream consumers are not broken.

`session/mod.rs:1281-1296` (json mode) and `:1299-1318` (stream-json mode) swap the read source from `session.total_tokens` → `session.accumulated_total_tokens.or(session.total_tokens)`. The `.or()` fallback is the load-bearing piece: legacy/empty sessions where `accumulated_*` was never populated still get the old behavior (which is wrong but at least not None). New `input_tokens` / `output_tokens` fields read `session.accumulated_input_tokens.or(session.input_tokens)` / `accumulated_output_tokens.or(session.output_tokens)`.

## Specific references

- `session/mod.rs:67-72` — the docstring explicitly names the bug source: "`Session::total_tokens` ... is the last-turn context size and gets overwritten / reset on compaction — see `update_session_metrics` in `goose/src/agents/reply_parts.rs`". This is exactly the right level of context for the next maintainer; without it, a future "simplify by reading total_tokens directly" PR would silently re-introduce the bug.
- `session/mod.rs:69, :91-95` — `#[serde(skip_serializing_if = "Option::is_none")]` on the two new fields preserves wire compatibility. Old consumers see `total_tokens` in the same name and position with a *correct* (larger) value; new consumers can opt into reading `input_tokens`/`output_tokens`.
- `session/mod.rs:1284` — `total_tokens: session.accumulated_total_tokens.or(session.total_tokens)`. The `.or()` semantics: returns `accumulated_total_tokens` if `Some`, else falls back to `total_tokens`. Correct for the migration path where older sessions on disk have `accumulated_*` as `None` but legacy `total_tokens` populated.
- `session/mod.rs:1299-1318` — stream-json restructure. Pre-fix, `total_tokens` was read in a one-line `.and_then(|s| s.total_tokens)` chain; post-fix, captures `let session = ...ok();` then matches `Some(s)` / `None` to triple-extract. Slightly more verbose but unavoidable given the three correlated reads.

## Risks / nits

- **No `cargo test` was run.** Pre-Submission checklist: `cargo check -p goose-cli` is checked, `cargo test -p goose-cli` is *unchecked* with comment "please run in CI". The new `JsonMetadata` and `StreamEvent::Complete` shape have no unit-test coverage in this PR — if there's an existing snapshot test for the stream-json wire shape, this PR would silently regress it (or, if the snapshot test simply re-runs and accepts the new shape, a future regression in the field omission semantics would not be caught). Maintainers should at minimum add a serde-roundtrip test asserting that `StreamEvent::Complete { total_tokens: Some(100), input_tokens: None, output_tokens: None }` serializes to `{"total_tokens":100,"status":"completed"}` (no `input_tokens`/`output_tokens` keys) and that `Some(0)` serializes as present (`skip_serializing_if = "Option::is_none"` correctly preserves zero).
- The fallback `.or()` chain assumes the `accumulated_*` fields exist on `Session` already — implicit dependency on whatever PR/commit populated those fields. If `accumulated_*_tokens` is not populated at all in current sessions (only on a forward path), the fallback runs every time and the fix appears to work in tests but doesn't actually move the needle in production. Worth confirming `accumulated_*_tokens` is being written by `update_session_metrics` or wherever the running sum lives.
- Manual testing claim ("re-ran a multi-turn `goose run --output-format stream-json` against our eval harness and confirmed the emitted `total_tokens` now matches") is good signal but un-reproducible from the diff. A small test fixture with two simulated turns would close that gap.

## Verdict

`merge-after-nits` — the fix is correct (right field source, right `.or()` fallback for migration, right `skip_serializing_if` for wire compatibility) and the JSDoc-equivalent struct doc is exactly the kind of guard against future re-regression. But the missing `cargo test -p goose-cli` run and the absence of a serde-roundtrip unit test on the new `Complete` shape are real gaps for a fix that touches a public CLI output format. Add a roundtrip test, run `cargo test`, then merge.
