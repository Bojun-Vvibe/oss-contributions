# openai/codex#20498 — Add a reusable log formatter

- **Author:** Rasmus Rygaard (rasmusrygaard)
- **Head SHA:** `6d2b55341b93334c1c7cb5432d546e5ac0438696`
- **Base:** `main`
- **Size:** +33 / -12 across 1 file
- **Files changed:** `codex-rs/state/src/log_db.rs`

## Summary

Single-file refactor that converts the previously-private `SpanFieldVisitor` and `MessageVisitor` structs in the `log_db` tracing layer into `pub` types with proper accessor methods (`thread_id()`, `message()`, `into_thread_id()`, `into_parts()`), so external crates can reuse the visitor logic for parsing tracing spans/events into structured log entries instead of having to re-implement the same field-extraction loops. The motivating change is at the call-site level — `on_new_span`, `on_record`, and `on_event` now use the new accessor methods (`visitor.into_thread_id()`, `visitor.into_parts()`) instead of touching the struct fields directly.

## Specific code references

- `log_db.rs:159`: `on_new_span` call-site flip — `visitor.thread_id` field access becomes `visitor.into_thread_id()` consuming-accessor call. Same in `on_record` at `:184` (where the value is also captured into a local `let thread_id = visitor.into_thread_id()` for later reuse). This is the right shape: a consuming accessor signals "I'm done with this visitor, give me ownership of the field" and is more idiomatic than direct field punching once the type is `pub`.
- `log_db.rs:195`: `on_event` is the most-changed call site. Old shape was `let thread_id = visitor.thread_id.clone().or_else(|| event_thread_id(event, &ctx))` (clone needed because the visitor's `message` field was about to be moved at `:208`). New shape is `let (message, visitor_thread_id) = visitor.into_parts(); let thread_id = visitor_thread_id.or_else(|| event_thread_id(event, &ctx))` — a single consuming destructure replaces the clone+later-move pair, giving slightly cheaper hot-path behavior (one fewer `String` clone per event) and clearer ownership semantics.
- `log_db.rs:241-258`: `SpanFieldVisitor` becomes `pub` with `#[derive(Debug, Default)]`, gains a borrowing accessor `pub fn thread_id(&self) -> Option<&str>` returning `as_deref()` and a consuming accessor `pub fn into_thread_id(self) -> Option<String>`. Both are present because external callers need either depending on whether they want to peek (borrow) or consume — exposes both halves of the standard "get-or-take" pattern.
- `log_db.rs:415-432`: `MessageVisitor` parallel treatment — `pub`, `#[derive(Debug, Default)]`, plus three accessors: `pub fn message(&self) -> Option<&str>`, `pub fn thread_id(&self) -> Option<&str>`, and the new consuming `pub fn into_parts(self) -> (Option<String>, Option<String>)` returning `(message, thread_id)`. The `into_parts` shape is exactly right because the fields are coupled — consumers almost always want both at once, and forcing two separate `into_*()` calls would either require interior mutability or make the second call impossible after the first consumes `self`.

## Reasoning

This is a clean preparatory refactor — no behavior change, no new tests required (the existing tracing-layer behavior is fully covered by the surrounding `Layer<S>` impl tests), and the diff is mechanical at every call site. The visitor-to-pub-with-accessors pattern is the right shape for tracing-subscriber field extractors that other crates legitimately need to reuse.

Three small notes: (a) the borrowing accessors (`thread_id() -> Option<&str>`, `message() -> Option<&str>`) are added but not used by any caller in this PR — they're there for the external consumer the PR is preparing for, but it's worth a one-line doc comment explaining "the consuming `into_*` variants are preferred for the common case where the visitor is dropped immediately after extraction". (b) `into_parts` returning a positional tuple `(Option<String>, Option<String>)` is slightly error-prone for a same-type pair — a named struct or two separate `into_message`/`into_thread_id` consuming accessors (with a `take_*` variant for partial consumption) would be safer at the use-site. (c) The PR body is the boilerplate "External (non-OpenAI) Pull Request Requirements" template — the actual change description is missing entirely; reviewers shouldn't have to read the diff to know what the PR is for.

## Verdict

**merge-after-nits**
