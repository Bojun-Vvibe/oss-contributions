# block/goose#8994 — feat(agent): detect repeated tool errors and inject recovery hint

- PR ref: `block/goose#8994` (https://github.com/block/goose/pull/8994)
- Head SHA: `68f16b33851cd64e7b9284e9d98ef28972e3bcbd`
- Title: feat(agent): detect repeated tool errors and inject recovery hint
- Verdict: **merge-after-nits**

## Review

The diagnosed gap is real and the fix targets it precisely. The existing
`RepetitionInspector` only catches identical *call signatures* (same
name + same arguments), so an agent that calls
`pup error-tracking issues get <id>` with a different ID each time but
gets the same 404 every time loops indefinitely. Tracking by
`(tool_name, error_text)` fingerprint instead of `(tool_name, args)` is
the right axis.

State management in `crates/goose/src/tool_monitor.rs` looks correct:
the `Mutex` interior is required by the `&self` trait bound on
`inspect()` (called this out explicitly in the PR description, which I
appreciate), and `record_error` only allocates new `String`s on
fingerprint change, so the steady-state cost per tool call is one
`Mutex::lock` plus a string compare — fine.

The integration in `crates/goose/src/agents/agent.rs:1530-1559` of the
diff handles all three result shapes: `Err(e)` → `record_tool_error`,
`Ok(r) if r.is_error == Some(true)` → record with extracted text (or
serialized content as fallback), `Ok(_)` → `record_tool_success`. The
text extraction at `agent.rs:1542-1547` correctly joins all text
chunks, which is what you want for fingerprinting — partial extraction
would produce false negatives where the same logical error reads as
different fingerprints.

Nits before merge:

1. **Threshold not configurable.** The PR description says "default: 3"
   but the diff appears to hard-code that constant inside the
   inspector. Worth surfacing it as a config knob (alongside the
   existing repetition threshold) so a session that legitimately
   retries — e.g. a polling tool that 404s while a resource is being
   provisioned — isn't penalized after exactly 3 polls.

2. **Error-text normalization.** Two errors that differ only in a
   timestamp, request ID, or `attempt 1/3` suffix will hash as
   different fingerprints and the inspector will never trip. If the
   author has a normalization pass it'd be worth calling out; if not,
   one obvious small win is stripping common UUID / ISO8601 patterns
   before fingerprinting.

3. **`FINDING_ID_REPEATED_ERROR` / `FINDING_ID_REPEATED_CALLS`
   constants** are good — the PR description mentions them explicitly.
   Worth one assertion in tests that the emitted `Deny` carries
   `finding_id: "REP-002"` so a future renumbering doesn't silently
   break downstream consumers that key off the finding ID.

Solid, focused fix. Ship after at least the configurability nit.
