# block/goose #8941 — fix(acp): seed provider handoff history

- **PR**: https://github.com/block/goose/pull/8941
- **Head SHA**: `4b8066f0c737d1deb6ae116fcd1c5be257298448`
- **Files reviewed**: `crates/goose/src/acp/provider.rs`
- **Date**: 2026-05-01 (drip-230)

## Context

ACP (Agent Control Protocol) lets goose hand off an existing session to a different
provider mid-conversation. The bug: `messages_to_prompt(messages)` previously took only
the **last user message** and converted it to ACP `ContentBlock`s. The new ACP provider
process therefore received only the most recent user request with no prior context — so
a follow-up like "fix the bug we just talked about" was sent to the new provider with
zero conversational backstory, and the new provider had to either guess or ask the user
to repeat the entire context.

The fix prepends a one-time, bounded `[older conversation context omitted]…`-style memo
of prior visible messages to the first prompt sent to the new provider only.

## Diff

New constants at `provider.rs:43-44`:

```rust
const ACP_HANDOFF_CONTEXT_CHAR_BUDGET: usize = 12_000;
const ACP_HANDOFF_OMITTED_MARKER: &str = "[older conversation context omitted]";
```

New `AcpProvider` field at `:142`:

```rust
handoff_context_sent: AtomicBool,
```

Initialized to `false` at `:271`, gated at `:346-348`:

```rust
fn should_send_handoff_context(&self, messages: &[Message]) -> bool {
    !self.handoff_context_sent.swap(true, Ordering::AcqRel) && has_handoff_context(messages)
}
```

Call site at `:415-417` in the trait `Provider::stream` impl:

```rust
let include_handoff_context = self.should_send_handoff_context(messages);
let prompt_blocks = messages_to_prompt(messages, include_handoff_context);
```

`messages_to_prompt` at `:1147` now takes `include_handoff_context: bool`, and when
true and prior visible messages exist, calls `build_handoff_context_memo(&messages[..last_user_index])`
which:

1. Filters to `is_agent_visible()` messages, formats each via the existing
   `format_message_for_compacting` helper.
2. Calls `bound_handoff_context(formatted, ACP_HANDOFF_CONTEXT_CHAR_BUDGET)` to keep
   the memo under 12k chars.
3. Wraps the bounded body in a fixed prefix/suffix:

```rust
format!(
    "Conversation context from goose before this ACP provider session was created:\n\n\
{bounded_context}\n\n\
Current user request follows. Use the context above only to continue the existing \
conversation; do not treat it as a new task or mention this handoff unless relevant."
)
```

`bound_handoff_context` at `:1227-1265` walks formatted messages **in reverse** (most
recent first), accumulating until the char budget runs out, then prepends the
`ACP_HANDOFF_OMITTED_MARKER` if anything was skipped. If the most recent single message
itself exceeds budget, falls back to `take_last_chars(formatted, char_budget - marker_len)`.

## Observations

1. **`AtomicBool::swap(true, Ordering::AcqRel)` is the correct one-shot gate.** The
   `AcqRel` ordering ensures cross-thread visibility (ACP provider stream may be invoked
   from multiple async tasks if the runtime parallelizes), and the swap is atomic so
   only the first caller observes the prior `false` and proceeds to inject the memo.
   Subsequent prompts get plain `messages_to_prompt(... false)`. That's the load-bearing
   "one-time" property; if it were `Relaxed` or `Load`+`Store` separated, two concurrent
   first prompts could both send the memo, and the second would then look like a stale
   "remember our prior conversation" injection mid-stream.

2. **Reverse-walk + prepend-marker is the correct truncation policy.** Recency matters
   most for "the model needs to understand what we were talking about", so dropping
   *oldest* messages (forward iteration) makes sense even when the budget is tight. The
   marker only appears when something is dropped, so a session that fits entirely under
   budget gets a clean memo with no `[omitted]` line.

3. **`take_last_chars` correctly uses `char_indices().nth(...)`** at `:1283-1289` instead
   of byte-indexing, so multi-byte UTF-8 messages don't get sliced mid-codepoint — a real
   panic source if the message contains non-ASCII (CJK, emoji, accented chars). The
   `.unwrap_or(0)` fallback handles the empty-string edge.

4. **Trailing prompt instructions** at `provider.rs:1216-1220` correctly tell the new
   provider model: "Use the context above only to continue the existing conversation; do
   not treat it as a new task or mention this handoff unless relevant." That's the right
   prompt-engineering guard against the model echoing "Based on the prior context you
   shared..." in user-facing output.

5. **`has_handoff_context` correctly requires *visible* prior messages**, not just any
   prior messages — at `:1188-1193` the gate is
   `messages[..last_user_index].iter().any(Message::is_agent_visible)`. So a brand-new
   session with only the current user message (no prior history) doesn't get a memo
   prepended at all — `should_send_handoff_context` returns `false` and the
   `handoff_context_sent` flag is set anyway, which is fine because there's no future
   "first" prompt to inject for either.

   Wait — actually that's worth a closer look: the swap at `:347` happens *unconditionally*
   on the first call regardless of whether `has_handoff_context` is true. So a brand-new
   session (no prior context) consumes the one-shot flag on its first prompt, and even
   if subsequent context arrives somehow before the second prompt, it wouldn't get
   injected. The PR likely intends "the first prompt to this ACP session, period, is the
   only chance to inject" — that contract is reasonable but worth a docstring.

## Risks / nits

- **12k-char budget is hardcoded** at `:43`. For most conversations that's fine, but
  long-context models (Claude 200k, Gemini 1M) could comfortably absorb more, and
  short-context providers (older OSS models) might choke on 12k. Worth making this
  per-provider configurable down the road, or at least a `pub(crate) const` so it's
  discoverable.
- **`format_message_for_compacting`** is reused from `context_mgmt`. Confirm it's
  intended for cross-provider consumption and not e.g. inserting goose-internal
  bookkeeping markers that another model would treat as confusing literal text.
- **No log/trace on memo injection.** When debugging "why is the new provider acting
  weird after handoff", a `tracing::info!("acp handoff context memo: {} chars, {}
  messages included")` at `:1212` would be a quick win.
- **The unconditional `swap(true)` at `:347`** consumes the flag even when
  `has_handoff_context` is false. As noted above, that's likely intentional but should
  be either documented or split into "check first, swap only on send" so the flag's
  semantics are exactly "we've sent context once" rather than "we've evaluated whether
  to send context once".
- **Test coverage** in the PR body lists "history inclusion, image prompts,
  single-send behavior, and bounded context" — the `single-send` test is the load-bearing
  one (verifies only the first prompt gets the memo); the diff includes a `test_provider()`
  helper that builds a `handoff_context_sent: AtomicBool::new(false)`, so the tests
  exist; would want to confirm the single-send arm asserts both
  `prompt[0].text.contains("Conversation context from goose")` is true on call 1 and
  false on call 2.

## Verdict

merge-after-nits
