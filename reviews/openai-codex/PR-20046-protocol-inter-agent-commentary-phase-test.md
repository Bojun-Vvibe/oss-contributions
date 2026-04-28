# openai/codex#20046 — test protocol: lock inter-agent commentary phase

- **Repo:** [openai/codex](https://github.com/openai/codex)
- **PR:** [#20046](https://github.com/openai/codex/pull/20046)
- **Head SHA:** `af9983fa9dfa92fa7a6a50a657c738e2d26fcec3`
- **Size:** +22 / -0 (one new test in `protocol.rs`)
- **State:** OPEN

## Context

Adds a regression test that pins the contract of
`InterAgentCommunication::to_response_input_item()` — specifically, that
the resulting `ResponseInputItem::Message` carries `phase:
Some(MessagePhase::Commentary)`. This is part of the inter-agent commentary
feature where one agent's communication to another is replayed into the
target agent's input as an *assistant* message that's tagged as commentary
(so it doesn't bleed into the next user-turn-completion logic).

## Design analysis

The test (`codex-rs/protocol/src/protocol.rs:4021-4040`) is well-constructed:

```rust
#[test]
fn inter_agent_communication_response_input_item_preserves_commentary_phase() {
    let communication = InterAgentCommunication {
        author: AgentPath::root(),
        recipient: AgentPath::root().join("reviewer").expect("recipient path"),
        other_recipients: vec![AgentPath::root().join("worker").expect("recipient path")],
        content: "review the diff".to_string(),
        trigger_turn: true,
    };

    assert_eq!(
        communication.to_response_input_item(),
        ResponseInputItem::Message {
            role: "assistant".to_string(),
            content: vec![ContentItem::OutputText {
                text: serde_json::to_string(&communication).expect("serialize communication"),
            }],
            phase: Some(MessagePhase::Commentary),
        }
    );
}
```

Three things this test gets right:

1. **Full structural equality, not field-by-field assertions.** The
   `assert_eq!` against a constructed `ResponseInputItem::Message` will fail
   loudly if anyone changes the `role`, the `ContentItem` variant, the
   serialization choice, *or* the phase. That's the right shape for a
   contract pin — we want this to break on any unintended drift, not just
   the phase regression.
2. **Reconstructs the expected content via the same `serde_json::to_string`
   call the production code uses.** This dodges the brittle "hardcoded
   string" alternative (which would break every time a field was added to
   `InterAgentCommunication` even if the change was harmless).
3. **Exercises a non-trivial author/recipient triple.** `author = root`,
   `recipient = root/reviewer`, `other_recipients = [root/worker]`,
   `trigger_turn = true`. That's enough complexity to catch any
   serialization shortcut that drops the `other_recipients` or coerces
   `trigger_turn` to a default.

The new test is appended after
`session_source_from_startup_arg_normalizes_custom_values` (line 4042),
which is the natural placement — both are protocol-shape regression pins in
the same `mod tests`.

## Risks / nits

1. **Test name is descriptive but verbose** —
   `inter_agent_communication_response_input_item_preserves_commentary_phase`.
   That's load-bearing for the failure message ("which test broke and what
   was it pinning?"), so I wouldn't shorten it. Optional shortening:
   `inter_agent_communication_to_response_input_preserves_commentary_phase`.
2. **No counter-example test.** A second test asserting that *non-commentary*
   contexts produce `phase: None` (or a different `MessagePhase`) would lock
   the discrimination boundary tighter — i.e., catch the bug "everything is
   now `Commentary`" which this test alone wouldn't surface. Optional, can
   be a follow-up.
3. **`trigger_turn: true` isn't asserted to do anything in the output.**
   The test sets it but the assertion on `ResponseInputItem::Message`
   doesn't depend on it. If the production code's `trigger_turn` later
   means "tag the message differently in the output," this test would
   silently pass. Documenting "trigger_turn does not affect
   to_response_input_item()" via a comment in the test would prevent that
   from rotting unnoticed.

## Verdict

**merge-as-is.** Tight, focused regression test with full structural
equality, no brittleness, and an obvious failure message if the contract
drifts. The companion build-fix PR (#20051) handles the trunk lint break
that this PR's signature changes triggered.

## What I learned

- For protocol-shape regression tests, asserting full structural equality
  (`assert_eq!(actual, fully_constructed_expected)`) catches more than
  field-by-field assertions and is *less* brittle if the expected value is
  built from the same serde call as production rather than a hardcoded
  string.
- A contract pin's test name doubles as failure documentation. Verbose names
  like `..._preserves_commentary_phase` are the right tradeoff because the
  CI failure log is often the only context the next author sees.
- When a test has an unused-looking field (`trigger_turn: true` here), a
  one-line comment about why it's there ("set to make the input realistic;
  does not affect output shape") prevents future bit-rot.
