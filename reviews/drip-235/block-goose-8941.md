# block/goose#8941 — fix(acp): seed provider handoff history

- **PR**: https://github.com/block/goose/pull/8941
- **Head SHA**: `4cd2dfc3e0631248c195d4aa3f4cd200a7bdffeb`
- **Size**: +261 / -25, 1 file (`crates/goose/src/acp/provider.rs`)
- **Verdict**: **merge-after-nits**

## Context

Two situations where the agent would lose conversation history:
1. Continuing a thread with an ACP provider after relaunching the app (ACP providers are deliberately configured not to store session history).
2. Switching a thread over to an ACP provider mid-stream.

Both cases left the ACP provider's first prompt seeing only the most recent user message via `messages_to_prompt` (which historically just grabbed `last_user`), so prior conversation context was invisible to the downstream agent.

## What's right

**`AcpProvider` gains `handoff_context_sent: AtomicBool` field at `:145` initialized to `false` at `:274`.**

One-shot flag — once the first prompt has been sent on this provider instance, no subsequent prompts re-prepend handoff context. This is exactly the "send context with the first prompt only" semantic the use case requires.

**`claim_handoff_context` at `:349-356` uses `swap(true, Ordering::AcqRel)` for atomic claim-and-set.**

The `AcqRel` ordering is right for the "I'm the first to consume this token" pattern — Acquire on read (we observe whatever was published before the swap), Release on write (subsequent observers see our `true`). The returned `HandoffContextClaim { first_prompt, include_context }` carries both bits separately — `include_context = first_prompt && has_handoff_context(messages)` — so the call site can distinguish "we claimed the slot but there's nothing to send" (no rollback needed) from "we claimed the slot and we did send context" (rollback on prompt-send-failure).

**`stream` method at `:419-440` — claim before, rollback on send-failure.**

```rust
let claim = self.claim_handoff_context(messages);
let prompt_blocks = messages_to_prompt(messages, claim.include_context);
// ... clear pending_tool_updates ...
let mut rx = match self.prompt(session_id, prompt_blocks).await {
    Ok(rx) => rx,
    Err(e) => {
        if claim.first_prompt {
            self.handoff_context_sent.store(false, Ordering::Release);
        }
        return Err(ProviderError::RequestFailed(...));
    }
};
```

The rollback is the load-bearing detail: if the very first prompt fails to send (e.g. ACP transport hiccup), the next call must re-attempt with handoff context, otherwise the user sees "I have no context" forever after one transient send error. Rolling back via `store(false, Ordering::Release)` is correct — `Release` ensures the rollback is visible to any subsequent `swap(true, AcqRel)`.

**`messages_to_prompt` rewritten at `:1161-1192` with new `include_handoff_context: bool` parameter.**

Previously: `find` last user message via reverse iteration, push its text/image content. New: `last_user_message_index` (using `rposition` at `:1196`), then optionally prepend the handoff memo, then push the latest user's content. The control flow is cleaner (`Some(last_user_index) else return`) and the behavior is unchanged when `include_handoff_context = false`.

**`has_handoff_context` at `:1201-1207`.**

```rust
fn has_handoff_context(messages: &[Message]) -> bool {
    last_user_message_index(messages).is_some_and(|last_user_index| {
        messages[..last_user_index].iter().any(Message::is_agent_visible)
    })
}
```

The "is there *any* agent-visible message before the latest user message?" predicate is the right minimal test — it skips the memo construction work entirely for fresh sessions (where the memo would be empty anyway) and avoids burning the `handoff_context_sent` flag on a no-op.

**`build_handoff_context_memo` at `:1209-1227`.**

Uses `format_message_for_compacting` (imported from `context_mgmt` at `:33`) to render each prior agent-visible message into the same shape used by the existing context-compaction codepath. Reusing the compaction formatter means handoff context and compaction summaries share a stable wire shape — good consistency, and any future formatter improvement applies symmetrically.

The wrapping prose is well-considered:

> `Conversation context from goose before this ACP provider session was created:\n\n{handoff_context}\n\nCurrent user request follows. Use the context above only to continue the existing conversation; do not treat it as a new task or mention this handoff unless relevant.`

The "do not treat it as a new task or mention this handoff unless relevant" instruction is the load-bearing prompt-engineering bit — without it, the downstream agent often verbally acknowledges the handoff which is confusing UX.

**Tests at `:1424-1602` (visible portion).**

`test_provider`/`test_provider_with_tx` factory at `:1432-1453` constructs a synthetic `AcpProvider` with `handoff_context_sent: AtomicBool::new(false)`, no real loop thread, no real `tx` — the right test scaffolding for testing the prompt-construction logic without spinning up an ACP transport.

The two visible test arms pin the contract:
- `messages_to_prompt_without_prior_history_preserves_current_prompt` — no prior history, `include_handoff_context = true`, only the current request comes through. Pins the "no false memo when there's nothing to prepend" semantics.
- `messages_to_prompt_prepends_handoff_context_before_latest_user` — multi-turn history (user prompt → assistant text + tool_request → user tool_response → user "continue from there"), prepended memo present.

## Risks / nits

- **`handoff_context_sent` is per-`AcpProvider` instance.** If a thread switches *back* to the same provider instance after switching to a different one, the memo will not be re-sent (the flag is already `true`). Probably correct semantics — the downstream agent already saw the context once on this instance — but worth a one-line comment confirming that's the intended invariant.
- **`AtomicBool` ordering choice (`AcqRel` for swap, `Release` for rollback) is correct but unannotated.** A one-line comment at `:355` explaining "AcqRel: we want to observe any state published by prior swaps and publish our own claim" would help future maintainers who don't speak C++11 memory-order fluently.
- **The "ACP transport send-failure rollback" path is not tested.** The visible test scaffolding constructs `tx: None` so `prompt(...)` would fail differently than the rollback path expects. A test that supplies a `tx` whose receiver is closed (so `send().await` fails), invokes `stream()`, asserts the error is propagated AND `handoff_context_sent.load(Acquire) == false`, would directly pin the load-bearing rollback at `:430-432`.
- **`format_message_for_compacting` reuse is good but couples the wire shape between handoff and compaction.** A future change to compaction formatting (e.g. adding token-budget truncation) will silently change handoff format too. Worth a one-line comment at `build_handoff_context_memo` documenting the intentional reuse.
- **No bound on memo size.** A long thread (hundreds of agent-visible messages) becomes a single very large `ContentBlock::Text` prepended to the prompt. Downstream model context window may then truncate the actual user request. Worth a follow-up: a max-token budget on the memo with explicit truncation at the head ("...earlier context truncated...") rather than letting the model see a randomly-truncated tail. Not blocking, but operationally important for long threads.
- **`include_handoff_context` parameter on `messages_to_prompt`** is positional `bool` — easy for a caller to flip the wrong way at a future call site. Either a named struct field or just a doc-comment at the call sites would prevent the foot-gun. The `claim.include_context` named-field call site is fine; a future direct caller is the risk.
- **`build_handoff_context_memo` returns `Option<String>`** but is called only when `has_handoff_context(messages) == true`, which already guarantees there is at least one agent-visible message. The `if formatted_messages.is_empty() { return None; }` guard at `:1219-1221` is therefore dead code in the current call graph. Harmless defense-in-depth, just call out so it doesn't get refactored away accidentally.

## Verdict

**merge-after-nits.** Real UX bug (silent context loss across ACP handoff is a confusing failure mode), surgical fix that lives entirely in the ACP provider's existing `messages_to_prompt` boundary, atomic-flag claim semantics correctly handle the send-failure rollback, and reuses the established `format_message_for_compacting` formatter for wire-shape consistency. The notes above are mostly observability and follow-up considerations (memo size cap, send-failure rollback test, ordering comments) — none block the merge.
