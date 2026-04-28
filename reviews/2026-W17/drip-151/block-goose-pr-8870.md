# block/goose#8870 — fix(cli): emit cumulative token usage in stream-json/json complete event

- **Repo:** [block/goose](https://github.com/block/goose)
- **PR:** [#8870](https://github.com/block/goose/pull/8870)
- **Head SHA:** `961bbe0c27698e6c82a5c732076d58d9362f0f7c`
- **Size:** +28 / -5 (one file: `crates/goose-cli/src/session/mod.rs`)
- **State:** OPEN

## Context

`goose run --output-format stream-json` emits a `complete` event whose
`total_tokens` field downstream consumers (eval harnesses, billing
dashboards) interpret as the **cumulative** token cost of the session.
The implementation read `Session::total_tokens`, which is actually the
**last turn's context size** — overwritten on every LLM round-trip and
reset to summary-output count after compaction (per
`update_session_metrics` in `crates/goose/src/agents/reply_parts.rs`,
referenced in PR body).

Concrete manifestation: a multi-turn session reporting
`total_tokens=17099` when the cumulative-provider-reported usage was an
order of magnitude larger.

## Design analysis

The fix at `crates/goose-cli/src/session/mod.rs:1273-1284` (and
symmetric block at `:1294-1313`) reads the *accumulated* fields with
fall-through to the legacy fields:

```rust
total_tokens: session.accumulated_total_tokens.or(session.total_tokens),
input_tokens: session.accumulated_input_tokens.or(session.input_tokens),
output_tokens: session.accumulated_output_tokens.or(session.output_tokens),
```

Three things done right:

1. **`Option::or` for fallback semantics.** `accumulated_*` is the
   correct value when present; the legacy field is the fallback for
   pre-existing sessions that predate accumulation tracking. `.or()`
   gives "use the new field if it's `Some`, else fall back" with no
   silent overwrites.
2. **`total_tokens` keeps its name and ordinal position** in
   `JsonMetadata` (`:67`) and `StreamEvent::Complete` (`:90`).
   Existing consumers see a *correct* number where they previously saw
   an under-count, no parser breakage. This is the "right way to fix
   a wrong number" — change the value, keep the schema.
3. **`input_tokens` / `output_tokens` are `Option<i32>` with
   `skip_serializing_if = "Option::is_none"`** (`:68-71`, `:91-94`).
   Old consumers parsing the JSON don't see the new fields at all when
   they're absent. New consumers can opt in. Schema-additive change.

The error path at `:1281-1284` correctly sets all three fields to
`None` rather than emitting `0` or some other sentinel — `None`
serializes to JSON `null` which downstream harnesses can treat as
"unknown" rather than mistaking for "actually zero tokens."

The stream-json branch at `:1293-1313` does the same destructuring via
`match session { Some(s) => (...), None => (None, None, None) }`. Right
shape — single source of computation, three values plumbed through.

## Risks / nits

1. **`accumulated_*_tokens.or(legacy)` is wrong if the accumulated
   field was *intentionally* zero.** `.or()` treats `Some(0)` as a
   valid value (good) but `None` as "unset" (also good). The risk is
   if there's any code path where `accumulated_total_tokens` could be
   `Some(0)` for a session that did do real work but the accumulator
   reset incorrectly. The PR doesn't address that risk explicitly.
   Worth a comment in the session-manager code explaining the
   accumulator's reset semantics — `.or()` is correct only if
   "accumulator reset to 0" is impossible after a real LLM round-trip.

2. **Compaction interaction unclear.** The PR description mentions the
   `total_tokens` reset on compaction
   (`update_session_metrics` reset to summary output count) is the
   bug. Is `accumulated_total_tokens` also reset on compaction, or
   does it preserve the pre-compaction sum + the post-compaction
   tokens? If it resets, this fix doesn't actually solve the multi-
   turn-with-compaction case. The PR description claims it works
   ("ran a multi-turn `goose run --output-format stream-json` against
   our eval harness and confirmed the emitted `total_tokens` now
   matches the cumulative provider-reported usage") but I'd want to
   see a test that exercises a session that *triggers compaction*
   mid-run and asserts the accumulated value reflects pre-compaction
   tokens.

3. **No new test added.** The PR body explicitly notes
   `cargo test -p goose-cli : please run in CI` — no new tests
   asserting the new field semantics. Given the bug is "the value
   silently means the wrong thing," a regression test that pins
   `complete.total_tokens` against a known-multi-turn fixture would
   be exactly the right shape. Without it, the next refactor
   reintroduces the under-count.

4. **Backwards-compat claim is fragile.** "`total_tokens` keeps its
   name and position" is true for the JSON schema. But existing
   consumers who *reasonably* assumed `total_tokens` was last-turn
   context will now see a 10× larger number and may ring billing
   alarms unexpectedly. This is the classic "fixing a long-standing
   bug breaks downstream code that was working around it" trap. A
   release-note flag ("`total_tokens` semantics fixed: now cumulative,
   was last-turn") would help users not be surprised.

5. **Symmetric error paths.** Both arms (json mode at `:1281` and
   stream-json mode at `:1304`) have nearly identical `(None, None,
   None)` fallback shape. A small helper
   `extract_token_metrics(session: Option<&SessionMeta>) -> (Option<i32>,
   Option<i32>, Option<i32>)` would dedupe and make the fallback
   policy live in one place.

## Verdict

**merge-after-nits.** The fix is small, the diagnosis is precise (the
field name was always misleading; the value was always under-counting
on multi-turn). Pre-merge: add at least one regression test (nit 3)
and clarify the compaction interaction (nit 2 — at minimum a code
comment in `reply_parts.rs` documenting whether
`accumulated_total_tokens` survives compaction). Nits 1, 4, 5 are
follow-ups but worth a release-note line for nit 4.

## What I learned

- A schema field whose *name* says "total" but whose *value* is "last
  turn" is a particularly cruel kind of bug — it works for single-turn
  sessions, silently under-counts for everything else, and downstream
  harnesses build dashboards on the wrong number for months before
  someone notices the order-of-magnitude gap.
- `Option::or` is exactly the right primitive for "prefer new field,
  fall back to legacy field" migrations — as long as the new field's
  `Some(0)` semantics match what the caller wants. When it doesn't,
  you need an explicit `match` instead.
- `skip_serializing_if = "Option::is_none"` plus same-name-same-
  position field semantics is the correct schema-additive shape for
  fixing a value while not breaking parsers. Old parsers see no new
  fields and the same field they always parsed; new parsers can pick
  up the additions.
