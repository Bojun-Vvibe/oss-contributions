# openai/codex PR #19432 — Add token usage to turn tracing spans

- **Repo:** openai/codex
- **PR:** [#19432](https://github.com/openai/codex/pull/19432)
- **Head SHA:** `c9b64eff3838e2200c612d139a7e7958738669b4`
- **Author:** charley-openai
- **Size:** +154/-8 across 5 files
- **Reviewer:** Bojun (drip-24)

## Summary

Trace-only change that records per-response token usage on
`handle_responses{otel.name="completed"}` spans and aggregate
turn-level token usage on regular `session_task.turn` spans.
Adds a regression test (`turn_and_completed_response_spans_record_token_usage`)
that drives a mock SSE response with a known usage block and
asserts the corresponding span fields are populated.

The implementation is small and well-targeted. The interesting
bit is the `remove_task` API change in `core/src/state/turn.rs`.

## Key changes

### `core/src/state/turn.rs` (+3/-3)

Signature change at line 88:

```rust
-    pub(crate) fn remove_task(&mut self, sub_id: &str) -> bool {
-        self.tasks.swap_remove(sub_id);
-        self.tasks.is_empty()
+    pub(crate) fn remove_task(&mut self, sub_id: &str) -> Option<(&'static str, bool)> {
+        let task = self.tasks.swap_remove(sub_id)?;
+        Some((task.task.span_name(), self.tasks.is_empty()))
     }
```

This is a contract change: the old API returned `true` if the
turn became empty (regardless of whether the removal hit
anything), the new one returns `None` if the sub_id wasn't there.
The single caller in `tasks/mod.rs` is updated to match — but
this means the caller now correctly distinguishes "I tried to
remove a task that wasn't tracked" from "I removed the last
task". The old code silently treated both as "drop the active
turn", which was a latent bug if a sub_id was double-removed
(would falsely drop the active turn on the second call).

The `&'static str` for the span name is right because
`Task::span_name()` returns a compile-time literal — no
allocation cost.

### `core/src/tasks/mod.rs` (+33/-7)

The new fields are declared at line ~353 in `info_span!(...)`:

```rust
codex.turn.token_usage.input_tokens = field::Empty,
codex.turn.token_usage.cached_input_tokens = field::Empty,
codex.turn.token_usage.non_cached_input_tokens = field::Empty,
codex.turn.token_usage.output_tokens = field::Empty,
codex.turn.token_usage.reasoning_output_tokens = field::Empty,
codex.turn.token_usage.total_tokens = field::Empty,
```

Then the population logic at line ~606 is gated on
`completed_task_span_name == Some("session_task.turn")` — i.e.,
*only* user-driven turns get the aggregate, not background
sub-agent turns or compaction turns. PR description confirms this
is intentional ("regular turn aggregate fields are restricted to
`session_task.turn`"). Good — keeps the analytics signal clean.

### `core/src/session/turn.rs` (+5/0)

Adds five `field::Empty` declarations at line 1915 on the
`try_run_sampling_request` span:

```rust
gen_ai.usage.input_tokens = field::Empty,
gen_ai.usage.cache_read.input_tokens = field::Empty,
gen_ai.usage.output_tokens = field::Empty,
codex.usage.reasoning_output_tokens = field::Empty,
codex.usage.total_tokens = field::Empty,
```

Note the namespace split: `gen_ai.usage.*` for the three OTel
GenAI semantic-convention fields (input/cache_read.input/output)
and `codex.usage.*` for the codex-specific reasoning + total
fields. This is the right call — `gen_ai.usage.reasoning_tokens`
isn't yet a stable OTel semantic convention, so emitting it under
the `codex.*` namespace avoids a future collision when OTel
standardizes it.

### `otel/src/events/session_telemetry.rs` (+17/0)

The actual recorder at line 305:

```rust
ResponseEvent::Completed { token_usage: Some(token_usage), .. } => {
    handle_responses_span.record("gen_ai.usage.input_tokens", token_usage.input_tokens);
    handle_responses_span.record("gen_ai.usage.cache_read.input_tokens", token_usage.cached_input());
    handle_responses_span.record("gen_ai.usage.output_tokens", token_usage.output_tokens);
    handle_responses_span.record("codex.usage.reasoning_output_tokens", token_usage.reasoning_output_tokens);
    handle_responses_span.record("codex.usage.total_tokens", token_usage.total_tokens);
}
```

The `cached_input()` method call (vs raw field access) tells me
there's a derived getter — confirms the reasoning_output_tokens
"surface what the runtime already collects" pattern that codex has
shipped a few times now (#19308, ollama #15768).

## Concerns

1. **Test asserts log-line substrings, not span attributes.**

   The new test (`turn_and_completed_response_spans_record_token_usage`
   at line 563 in `core/tests/suite/otel.rs`) asserts:

   ```rust
   line.contains("gen_ai.usage.input_tokens=3")
   ```

   This depends on `tracing_subscriber::fmt` rendering. If
   somebody changes the formatter (e.g. switches to JSON), the
   test breaks for non-functional reasons. Worth using the OTel
   `InMemorySpanExporter` shape if codex already has it
   elsewhere; otherwise this is fine for now.

2. **`completed_task_span_name == Some("session_task.turn")` is
   a stringly-typed gate.**

   The check at line ~606 compares `&'static str` for equality.
   If somebody renames the span in `Task::span_name()` for a
   refactor, this gate silently stops firing and the aggregate
   stops getting recorded — with no test failure unless they also
   update the test. A `const SESSION_TURN_SPAN_NAME: &str =
   "session_task.turn"` referenced from both sites would catch
   this at compile time.

3. **`remove_task` API change is correct but downstream callers
   should be audited.**

   The diff shows one caller updated (in `tasks/mod.rs`), but
   `pub(crate) fn` means there could be other in-crate callers.
   I'd want to grep for `.remove_task(` in `core/src/` to
   confirm there's only one — if there are more, they need the
   same `Option<(...)>` destructure.

4. **PR description notes "10 unrelated tests failed" in
   `cargo test -p codex-core`.**

   The author flagged this honestly. Worth confirming none of
   those failures are masking a regression caused by the
   `remove_task` semantics change (the "active turn dropped on
   double-remove" scenario could plausibly affect
   request-permissions or subagent-metadata-persistence tests).

## Verdict

`merge-after-nits` — clean trace-only PR with the right
namespace split (`gen_ai.*` for stable, `codex.*` for not-yet-
stable conventions), and the `remove_task` API change is actually
a latent bug fix. Three small follow-ups:

- audit other `remove_task` callers in `core/src/`,
- factor `"session_task.turn"` into a `const`,
- confirm the 10 unrelated test failures aren't actually related.

## What I learned

When you're adding observability to a runtime, the
`field::Empty` declaration up front + `current_span.record(...)`
later is the idiomatic `tracing` pattern, and the alternative
(creating a child span for the recording) loses the
attribute-on-the-original-span semantics that OTel collectors key
off. The `gen_ai.usage.*` vs `codex.usage.*` split is also
worth internalizing — emitting non-stable semantic-convention
fields under your own namespace gives you an upgrade path when
the standard catches up, instead of forcing a backward-
incompatible field rename later.
