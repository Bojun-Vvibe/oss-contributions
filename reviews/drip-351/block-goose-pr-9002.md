# block/goose PR #9002

- **Title**: feat(agent): auto-nudge model when it produces text without tool calls
- **Author**: enilsen16
- **Head SHA**: `1997569a92ba9167f1610009f60be766c835f425`
- **Diff**: +37 / -17

## Summary

When the model produces a turn with text but no tool calls (a "stalled" turn) and there are queued messages to send, inject a synthetic user message ("Please continue by using the available tools to accomplish the task.") up to `MAX_TEXT_ONLY_NUDGES = 3` times before falling back to existing retry logic. Counter resets on any turn that does call tools.

## File-by-file

### `crates/goose/src/agents/agent.rs`

- **L9-11**: constants `MAX_TEXT_ONLY_NUDGES = 3` and the canned nudge string. Reasonable defaults; would be nicer as agent config eventually but fine to ship as constants.
- **L19**: `consecutive_text_only_turns: u32` initialized at the top of `reply()`. Correct scope — tied to the agent loop, not the session.
- **L27-29**: counter reset on `!no_tools_called` happens *before* the `if no_tools_called` block. Correct ordering — a successful tool call always clears the streak, even if a previous nudge was injected.
- **L46-54 nudge injection**: the gate `!messages_to_add.is_empty() && consecutive_text_only_turns < MAX_TEXT_ONLY_NUDGES` is the load-bearing condition.
  - `messages_to_add.is_empty()` check is questionable: it gates nudging on whether the *current iteration* has buffered messages. If the first stalled turn produced no messages_to_add, we fall straight through to retry logic without ever nudging. Comment doesn't explain why this gate is there; suspect it's a guard against nudging when there's literally nothing to send back. Worth a comment.
  - Counter increments *before* the nudge yields (L47), so `MAX_TEXT_ONLY_NUDGES = 3` means at most 3 nudges. Correct.
- **L52-54**: nudge is both pushed to `messages_to_add` and yielded as `AgentEvent::Message(nudge)`. Yielding to the event stream means the UI will display the synthetic user message — good for transparency, users see why the model "kept going". Worth confirming this is desired UX (vs. a silent server-side push).
- **L55-86**: the retry-logic fallback branch is now nested inside the `else`. Counter is reset at L56 before retry — important so retry attempts don't permanently disable nudging next turn. Indentation is the only real cost of the refactor.

## Risks

- Nudge text is hardcoded in English. Non-English users will see this verbatim. Consider i18n hook or making the string configurable.
- 3 nudges with no token budget check — if model is stuck, this adds 3 extra (stalled-turn × full-context) round trips. For a small/cheap model this is fine; for an expensive one this is real money. A token-budget guard would be defensible.
- No new tests in the captured diff. A unit test that asserts (a) counter resets on tool-call turn, (b) max 3 nudges, (c) retry path engages on the 4th would lock the contract.

## Verdict

**merge-after-nits**
