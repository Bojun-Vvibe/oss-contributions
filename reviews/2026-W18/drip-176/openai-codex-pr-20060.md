# openai/codex#20060 — Add reasoning effort to turn tracing spans

- PR: https://github.com/openai/codex/pull/20060
- Head SHA: `99b39b63507df3372cd2dfa8f094c37faaf1fad6`
- Author: charley-openai
- Base: `main`
- Size: +72 / -23 across 4 files

## Summary

Follow-up to #19432 (which added token usage to turn/response spans). Adds
two new OpenTelemetry span fields so traces can be filtered by reasoning
effort:
- `codex.turn.reasoning_effort` on the turn span (`tasks/mod.rs:377`)
- `codex.request.reasoning_effort` on the `handle_responses` span
  (`session/turn.rs:1868`)

The value comes from the new helper
`TurnContext::effective_reasoning_effort_for_tracing()` at
`turn_context.rs:120-130`, which returns:
- the explicit `reasoning_effort` if set
- else the model's `default_reasoning_level`
- else literal `"default"`
- and `"default"` unconditionally if the model doesn't support reasoning
  summaries

## Notable design choices

- Span field is registered as `field::Empty` at `tasks/mod.rs:377` and
  recorded later at `tasks/mod.rs:665-668` *only when
  `records_turn_token_usage_on_span` is true*. This is consistent with
  the existing token-usage fields registered immediately after — same
  emit conditions, same record site, no risk of an empty-then-recorded
  race.
- `effective_reasoning_effort_for_tracing()` is a separate method from
  whatever computes effort for actual API calls. Good — tracing string
  shape ("default" sentinel for unsupported models) is a display
  concern that shouldn't leak into the request path.
- `turn.rs:1868` uses `%reasoning_effort` (Display) in the `tracing`
  macro, then the function returns `String`, so the macro borrows the
  string slice for the span's lifetime. Correct.
- Test update at `otel.rs:601-633` migrates the test from `Op::UserInput`
  (which has no `effort` field) to `Op::UserTurn` (which does), passing
  `effort: Some(ReasoningEffort::High)`. Then asserts
  `codex.request.reasoning_effort=high` appears in both
  `handle_responses` and `turn` log lines.

## Concerns

1. **`turn_context.rs:121` — `if self.model_info.supports_reasoning_summaries`
   gate.** This conflates two different concepts: "does the model
   *support* reasoning effort as a configurable parameter" vs "does the
   model emit reasoning *summaries* in its response stream." A model
   could conceivably support effort but not summaries (or vice versa).
   If `supports_reasoning_summaries` is the wrong gate, the trace will
   show `"default"` for models that *did* receive an explicit effort —
   silently misleading observability. Verify this is the right field
   on `ModelInfo`; if there's a `supports_reasoning_effort`, prefer it.
2. **String allocation per-turn.** `effective_reasoning_effort_for_tracing()`
   returns `String` (allocating). It's called twice per turn — once at
   `turn.rs:1856` for the `handle_responses` span, once at
   `tasks/mod.rs:664` for the turn span. That's two allocations per
   turn just for the trace label. Consider returning `&'static str` for
   the four `ReasoningEffort` variants + `"default"` (all of which are
   compile-time-known) or use `Cow<'static, str>` if the variant set
   ever needs a dynamic case.
3. **Sentinel `"default"` clashes with possible future
   `ReasoningEffort::Default`.** If OpenAI ever ships an explicit
   `Default` reasoning effort variant, `effort.to_string()` would
   produce `"default"` (lowercase from Display) and collide with the
   sentinel — a trace consumer can't distinguish "explicit default" from
   "no effort configured." Use a non-overloaded sentinel like
   `"unspecified"` or `"none"`.
4. **Test only covers `high`.** The test at `otel.rs:611-625` exercises
   exactly one branch (`Some(ReasoningEffort::High)`). The other three
   cases — `default_reasoning_level` fallback, `"default"` sentinel for
   unsupported models, and explicit `Low`/`Medium`/`High` variants —
   are uncovered. At minimum add a parameterized test or two more cases
   for the fallback and the unsupported-model branches, since those are
   the branches most likely to silently regress.
5. **`tasks/mod.rs:664-668` records `reasoning_effort` *and* token usage
   in the same `if records_turn_token_usage_on_span` block.** If the
   semantics ever diverge (we want effort recorded even when token
   usage isn't, or vice versa), this coupling will need to be
   unwrapped. Minor; just worth a comment at line 663 noting the
   shared gate.

## Verdict

**merge-after-nits** — small, targeted observability addition with a
test. The `supports_reasoning_summaries` gate (item 1) is the only
substantive concern — if that's the wrong field, the new trace data is
silently inaccurate for a subset of turns and you've made debugging
*harder* not easier. Items 2-5 are polish. Hold for clarification on
item 1, then ship.
