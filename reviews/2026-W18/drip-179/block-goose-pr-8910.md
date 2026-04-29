# block/goose#8910 — fix(cli): report cumulative total_tokens in stream-json/json output

- **PR**: https://github.com/block/goose/pull/8910
- **Head SHA**: `0b54c1d2a312de1cec0478df35650310d22ca94b`
- **Size**: +199 / −3, 2 files
- **Files**: `crates/goose-cli/src/session/mod.rs` (+3/-3), `crates/goose/tests/agent.rs` (+196/-0)
- **Fixes**: #8871

## Context

`--json` and `--stream-json` output modes were emitting `session.total_tokens` as `complete.total_tokens`, but `total_tokens` only stores the *last turn's* usage — so the value would reset on every chunk to the current turn's count rather than the cumulative session total. The session struct already has a separate `accumulated_total_tokens` field that correctly accumulates across turns (computed in `update_session_metrics` in `crates/goose/src/agents/reply_parts.rs`). The fix is a 3-line swap to read the right field in three CLI sites, plus a 196-line integration test that exercises the cumulative behavior across three reply turns.

## Why this exists / problem solved

Per #8871, downstream consumers of `--json` / `--stream-json` (CI scripts, billing dashboards, multi-step agent orchestrators) were getting per-turn token counts instead of session totals, breaking any "show me the cost of this entire session" workflow. The bug was a straightforward field-name mistake in three CLI emit sites.

## Design analysis

### The 3-line CLI fix

At `crates/goose-cli/src/session/mod.rs:1268`, `:1289`, and `:1448`:

```rust
// :1268 (JsonMetadata for --json mode)
- total_tokens: session.total_tokens,
+ total_tokens: session.accumulated_total_tokens,

// :1289 (StreamEvent::Complete for --stream-json mode)
- .and_then(|s| s.total_tokens);
+ .and_then(|s| s.accumulated_total_tokens);

// :1448 (get_total_token_usage for consistency)
- Ok(metadata.total_tokens)
+ Ok(metadata.accumulated_total_tokens)
```

This is exactly the right shape of fix. Three independent surfaces all read the same wrong field; all three swap to the right field. The `--json` emit at `:1268` is the one-shot mode (single complete payload at end); the `--stream-json` emit at `:1289` is the per-chunk-then-complete mode; `get_total_token_usage` at `:1448` is the in-process Rust API used by other goose-cli code paths. Touching all three keeps the field semantics consistent across surfaces.

One subtle correctness check: at `:1289`, the `session_manager.get_session(&self.session_id, false).await.ok().and_then(|s| s.accumulated_total_tokens)` pattern is unchanged in shape — only the field accessor swaps. The `false` arg presumably means "don't include messages" (the second bool to `get_session`); reading just metadata is the right scope here, no unnecessary deserialization of the message log.

### The 196-line test

`crates/goose/tests/agent.rs:1094-1289` adds `mod cumulative_token_tests` with one integration test `test_accumulated_total_tokens_across_multiple_turns`. The test:

1. Defines a `FixedUsageProvider` mock at `:1124-1176` that reports `(input=10, output=5, total=15)` on every `stream()` call.
2. Creates a real `SessionManager` against a `tempfile::tempdir()` so persistence behaves realistically.
3. Builds an `Agent` with a `Hidden` session (no on-disk session-file persistence beyond what the manager does in-temp).
4. Runs three replies (Turn 1, Turn 2, Turn 3) with `max_turns: Some(1)` each (each reply is one provider call → 15 tokens).
5. After each reply, fetches the session metadata and asserts:
   - After turn 1: `accumulated_total_tokens == Some(15)`.
   - After turn 2: `accumulated_total_tokens == Some(30)`.
   - After turn 3: `accumulated_total_tokens == Some(45)`.
6. Final assertion at `:1281-1284`: `total_tokens == Some(15)` — pins the *non-cumulative* semantics of the original `total_tokens` field, locking in that "last turn only" is the intended meaning.

This is solid integration-test discipline. It exercises the real `SessionManager`, the real `Agent.reply` loop, and the real `update_session_metrics` accumulation path — not a unit test of the field swap. If anyone in the future "fixes" `accumulated_total_tokens` to behave like `total_tokens` (or vice versa), this test catches it. The final assertion that `total_tokens == 15` (not `45`) is particularly valuable because it documents the *contract* of the two fields, not just the cumulative one.

A minor test-shape note: the test creates a `FixedUsageProvider` with `ProviderDef` and `Provider` impls (~50 lines of mock scaffolding at `:1124-1176`). If `goose/tests` already has a generic `MockProvider` with parameterizable usage, this is duplicative. A quick `rg 'impl Provider for' crates/goose/tests/` would surface it. If one exists, swap to it; if not, the `FixedUsageProvider` is a useful addition that future tests can reuse — consider promoting it to a shared `tests/common/mock_provider.rs`.

### What's not changed

The PR doesn't touch:

- `update_session_metrics` in `crates/goose/src/agents/reply_parts.rs` — correctly identified in the PR body as the source of truth for `accumulated_total_tokens`. No change needed there; it was already correct.
- The `total_tokens` field itself or its semantics — the PR explicitly preserves "last turn only" for that field, which is the right call (some downstream code may legitimately want last-turn usage, e.g., for displaying "this turn cost X tokens" in interactive mode).
- The TUI/interactive-mode rendering of token usage — those probably render `total_tokens` correctly because they're rendered per-turn. The PR is correctly scoped to the JSON output paths only.

## Risks

- **Backward-compat break for downstream JSON consumers.** Any tool currently parsing `complete.total_tokens` from `--stream-json` output and treating it as "the most recent turn's usage" will silently start receiving cumulative values after this fix. For users whose workflows are "ignore the value or sum it themselves," no impact. For users who were summing all the per-chunk values to get a session total (which is wrong but plausible if you didn't read the docs carefully), they'll now double-count. CHANGELOG entry needed; the PR body doesn't mention it.
- **The `Hidden` session type interaction.** The test uses `SessionType::Hidden`. If hidden sessions persist metadata differently from `Normal` sessions — e.g., `accumulated_total_tokens` updates only fire for non-hidden sessions — the test would pass but the `--json` mode (which is typically used with `Normal` sessions) might still be broken. The PR body doesn't say which session type the bug was reported against. Worth a second test case using `SessionType::Normal` to lock in that the fix works for the production-relevant case.
- **`get_total_token_usage` is a public API surface** (per the `pub async fn` at `:1444`). Any external consumer of `goose-cli` as a crate that calls `get_total_token_usage()` and treated the return as "last turn" will see semantic drift. This is probably empty (it's CLI-internal) but worth a `rg 'get_total_token_usage'` across consumers if any exist.
- **No floor-zero on `accumulated_total_tokens`.** If the upstream `update_session_metrics` ever emits a corrupt session with negative accumulated tokens (shouldn't happen but provider quirks exist), the JSON output will faithfully relay it. This is the right behavior — JSON should be honest — but a `>= 0` sanity assertion in the test would lock in the expected positive-monotone property.

## Suggestions

1. **Add a `Normal` session-type test case** (or change the existing test to `Normal`). The PR's test uses `Hidden`, and while the accumulator logic shouldn't differ by session type, the production usage of `--json` is overwhelmingly with non-hidden sessions. 5-line addition; locks in production-path correctness.
2. **CHANGELOG entry** for the JSON-output behavior change. Something like:
   > `--json` and `--stream-json` now report cumulative session token usage in `complete.total_tokens` instead of the last turn's usage (#8871). Downstream tools that summed per-chunk values will now double-count; switch to using the final `complete` event's value directly.
3. **Promote `FixedUsageProvider` to `crates/goose/tests/common/`** if not already present elsewhere. Future cost/usage tests will benefit. Defer to maintainer's preference on whether the 50-line mock belongs in the shared spot.
4. **Add a monotonicity assertion** in the test: between turns, `accumulated_total_tokens` should never decrease. A `>=` check between successive `session_after_N.accumulated_total_tokens` values is two lines and pins the property explicitly.
5. **Sanity-check the `--stream-json` per-chunk emission** — the bug was that every chunk was emitting the per-turn value. Does the new code emit `complete.total_tokens` only at the *end* of the session, or at every chunk? Looking at `:1289`, `emit_stream_event(&StreamEvent::Complete { total_tokens })` is inside what's presumably the on-stream-end branch (the field is named `Complete`), so this should be fine, but worth a confirmation read of the surrounding loop. If `Complete` is emitted per-chunk, the cumulative value would still update reasonably (it monotonically grows) but the event semantics would be misleading.
6. **Document the field semantics** in a doc-comment on the `Session` struct: `total_tokens: per-turn last-known usage; accumulated_total_tokens: cumulative since session start.` Future contributors will pick the right one without having to read the bug history.

## Verdict

**merge-after-nits** — the fix is correct, narrowly scoped, and the integration test is genuinely good (real SessionManager, real Agent.reply loop, three turns, asserts both fields' contracts). Asks: (1) flip the test to `SessionType::Normal` to cover production paths, (2) CHANGELOG entry calling out the backward-compat shift in JSON output semantics, (3) two-line monotonicity assertion. None are blockers; the JSON-consumer break is real but it's the right break (the previous behavior was the bug).

## What I learned

The "two semantically distinct fields with confusing names" anti-pattern (`total_tokens` vs `accumulated_total_tokens`) is exactly the shape that produces year-long unfixed bugs like this one. The reader of the CLI emit code at `:1268` had a 50/50 chance of picking the right field, and the type system gave no help (both are `Option<i32>`). The right defensive design — applied retroactively here in the form of a contract-pinning test — is to add a `// total_tokens: last turn only; for cumulative session usage, see accumulated_total_tokens` doc-comment at the field definition site. Better yet, rename for clarity: `last_turn_tokens` and `cumulative_tokens` would have made the bug visually obvious at the call site. This PR fixes the immediate bug correctly, and the integration test prevents regression, but the underlying naming hazard is still there for the next field consumer to trip over. A follow-up PR to rename the fields (with a deprecation alias) would be the durable fix; this PR is the right *first* step.
