# block/goose #8870 — fix(cli): emit cumulative token usage in stream-json/json complete event

- URL: https://github.com/block/goose/pull/8870
- Head SHA: `2f60a434a3bf4586bb153b289095d91b2ea6cc0d`
- Verdict: **merge-as-is**

## Review

- 36/-5 line surgical fix at `crates/goose-cli/src/session/mod.rs` switches the `total_tokens` field reported in both the JSON-output mode and the stream-json `Complete` event from `session.total_tokens` (last-turn context size, gets overwritten and *reset on compaction*) to `session.accumulated_total_tokens.or(session.total_tokens)` — closes a real bug where users running `goose run --output-format stream-json` and downstream-consuming the `Complete` event for cost/usage tracking saw a value that bore no relation to actual session token usage (compaction would zero it mid-session, fast turns showed only the small final context not the cumulative spend).
- The `.or(session.total_tokens)` fallback at `:1284,:1314` is load-bearing for sessions that predate the `accumulated_total_tokens` field — preserves backward-compat for resumed old sessions where only `total_tokens` is populated. Correct precedence: prefer cumulative when present, fall back to last-turn count when not.
- Adds matching `input_tokens` / `output_tokens` fields with `#[serde(skip_serializing_if = "Option::is_none")]` at `:73-76,:96-99` — the skip-if-none is critical because external consumers parsing the JSON shouldn't see new fields in the output for sessions that don't carry the data (avoids spurious `null` fields appearing in the `Complete` event for sessions that haven't yet been re-recorded with the per-side breakdowns). `Complete` event variant adds the same two fields with the same skip rule so the JSON shape extension is purely additive.
- The block comment at `:67-71` is the most important piece of the diff for future maintainers — explicitly names the trap (`Session::total_tokens` is "the last-turn context size and gets overwritten / reset on compaction — see `update_session_metrics` in `goose/src/agents/reply_parts.rs`") with a cross-reference to the function responsible for the reset. This is the kind of comment that prevents the bug from being reintroduced by a well-intentioned cleanup PR. Same comment shape repeated at `:93-95` for the `StreamEvent::Complete` variant. Honest fix, well-commented.
