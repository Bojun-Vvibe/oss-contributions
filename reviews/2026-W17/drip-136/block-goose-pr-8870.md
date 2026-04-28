# block/goose #8870 — fix(cli): emit cumulative token usage in stream-json/json complete event

- PR: https://github.com/block/goose/pull/8870
- Head SHA: `961bbe0c2769`
- Diff: 28+/5- across 1 file (`crates/goose-cli/src/session/mod.rs`)
- Base: `main`

## Verdict: merge-after-nits

## Rationale

- **Closes a real "stream-json consumers see only the last-turn token count" bug.** Today, `JsonMetadata` and the `Complete` stream event both populate `total_tokens` from `session.total_tokens` — which on a multi-turn session reflects only the *last* completed turn's usage, not the cumulative session total. Downstream tooling counting spend or budget per `goose run` invocation has been silently undercounting any session that ran more than one turn. The PR switches both call sites at `session/mod.rs:1276-1278` and `:1300-1302` to `session.accumulated_total_tokens.or(session.total_tokens)` — preferring the accumulator when present, falling back to the per-turn field when the session manager hasn't populated the cumulative cell yet (e.g. older session DB rows pre-dating the accumulator field). That fallback is the right shape: it preserves the prior behavior for not-yet-migrated sessions while picking up the correct cumulative count for new ones.
- **Symmetric expansion to `input_tokens` / `output_tokens`.** Lines `:67-71` (the `JsonMetadata` struct) and `:90-94` (the `Complete` stream event) both grow `input_tokens: Option<i32>` + `output_tokens: Option<i32>` fields with `#[serde(skip_serializing_if = "Option::is_none")]` — which is the right default for a wire-format addition (consumers that don't know the new fields stay backward-compatible; consumers that read them get them when present). Both new fields use the same `accumulated_*.or(*)` fallback pattern, so the input/output split is consistent with the total.
- **Error path (the `Err(_) =>` arm at `:1284-1289`) emits `None` for all three counts.** That's the right shape — when the session can't be loaded, surfacing `null` for the counts is honest about the unknown state, rather than pretending zero. With `skip_serializing_if`, the JSON output simply omits the fields when the session lookup fails, which is the correct contract.
- **Stream-json branch refactor at `:1294-1313` is structurally cleaner.** The prior code did `.and_then(|s| s.total_tokens)` inline; the new code captures the full `session: Option<Session>` at `:1294-1300`, then matches once at `:1301-1308` to extract all three fields with the fallback pattern, then constructs the `Complete` event at `:1309-1313`. That's one read of `get_session(...)` instead of (potentially) three, and the unified match expression makes the "all three or all None" invariant visible at the call site.

## Nits / follow-ups

- **The `accumulated_*.or(*)` fallback is a soft contract that should be locked down with a test.** A unit test on `Session` with `accumulated_total_tokens: Some(500)` and `total_tokens: Some(100)` asserting the JSON output emits 500 (not 100), plus the inverse with `accumulated_*: None`, would lock the fallback semantics. Without it, a future refactor that swaps the operands on the `.or(...)` would silently halve the reported usage.
- **No CHANGELOG/docs surface for the new `input_tokens`/`output_tokens` fields.** Stream-json consumers (CI dashboards, billing tools) need to know the new fields are coming so they can opt in. Worth a one-line release note.
- **`skip_serializing_if = "Option::is_none"` is correct but means the field's *absence* is ambiguous** — it could mean "session lookup failed" or "session existed but had no input/output split tracked". For dashboards relying on monotonic counters, the distinction matters. Worth a docstring on `JsonMetadata` documenting that absence ≡ "unknown", and downstream tooling should treat missing-but-`status=completed` as a soft signal to re-fetch on the next event.
- **`get_session(&self.session_id, false)` is called at `:1300` with `false` for some "include history" flag (inferred from context).** If the cumulative counters are only populated when history is included, this could return `None` for the accumulators. Worth confirming in the session manager that `accumulated_*_tokens` is populated independent of the history-inclusion flag.

## What I learned

The "per-turn vs cumulative" choice on a `total_tokens` field is exactly the kind of silent-undercounting bug that's invisible until someone reconciles a billing report. The right fix here is structural — *add* the cumulative field and prefer it, don't *swap* what the existing field means — because downstream consumers may have built tolerance around the prior semantics. The `accumulated_*.or(*)` fallback is the canonical way to migrate a wire-format field without a hard cutover. The lesson: any field named "total" in a streaming-protocol message should be unambiguous about whether it's per-message, per-turn, per-session, or per-billing-period — and the fallback pattern here is what migration looks like when the old field's semantic was wrong.
