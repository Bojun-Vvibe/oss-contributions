---
pr: 19082
repo: openai/codex
sha: b813aea26edb7cd14a91e5a4e81a9f3678e96e3e
verdict: merge-as-is
date: 2026-04-25
---

# openai/codex#19082 — Drop duplicate contiguous user messages during compaction

- **URL**: https://github.com/openai/codex/pull/19082
- **Author**: friel-openai

## Summary

`collect_user_messages()` is the function that decides which user-text
messages survive into the post-compaction summary prompt. Before this
PR it was a flat `filter_map().collect()` that kept *every*
non-summary user message verbatim, which means: (a) empty user
messages (e.g. from a bare Enter, or from a synthesized empty turn)
got carried into the summary, and (b) accidentally-doubled user turns
("send", click again, "send" — both go into the rolling window) got
counted twice during compaction, inflating the token budget assigned
to the user-history slice and biasing the summarizer toward whatever
phrase the user happened to repeat.

## Reviewable points

- `codex-rs/core/src/compact.rs:393` — the function rewrite from
  `filter_map().collect()` to a stateful loop with
  `previous_message: Option<String>`. The interesting choice: the
  state is reset to `None` (not just kept) whenever a non-user item
  is encountered, via the `let Some(message) = message else { previous_message = None; continue; }`
  branch. This implements "drop *contiguous* duplicates only" — the
  test at compact_tests.rs:185 asserts exactly this: three `repeat`
  user messages with an assistant message between #2 and #3 collapse
  to `["repeat", "repeat"]`, not `["repeat"]`. That's the right
  semantics; collapsing across an assistant turn would erase
  legitimate "user asked X, assistant answered, user asked X again"
  follow-ups.

- The empty-message check (`if message.is_empty() { continue; }`) is
  placed *after* the dedup-state assignment is skipped — i.e. an
  empty message neither gets emitted nor resets `previous_message`.
  That looks intentional and correct: an empty user turn shouldn't
  break a "user said X / [blank turn] / user said X again" dedup
  chain. Worth a one-line comment though, because at first read it
  looks like a bug: if `previous_message` survives an empty turn,
  what counts as "contiguous"? Answer: contiguous in *non-empty*
  user-messages, ignoring empties. That's defensible but not
  obvious from the code.

- `compact_tests.rs:128` — the new test
  `collect_user_messages_drops_contiguous_duplicates_and_empty_messages`
  exercises both new behaviors in one fixture (empty leading message,
  duplicate pair, assistant interrupt, repeated `repeat`, trailing
  empty). Good single-fixture coverage. The expected result
  `vec!["repeat", "repeat"]` is correct given the documented
  semantics.

- One minor: `previous_message: Option<String> = Some(String::new())`
  on line 396 is initialized to `Some("")` rather than `None`. This
  is a deliberate choice — it means a leading non-empty user message
  is *not* treated as "matches the previous one" because `""` won't
  equal it; *and* it means an empty leading message would be
  short-circuited by the `is_empty()` check before it could
  contaminate the dedup state. Net behavior is correct, but
  initializing to `None` and adjusting the comparison would be
  slightly easier to read. Not blocking.

## Rationale

The bug is real (empty + duplicate user turns inflate the
post-compact summary input), the fix is local to one function, the
test pins the exact semantic that matters most (don't collapse
across non-user items), and the PR ships only those changes plus the
test. Mergeable.

## What I learned

Compaction is the one place in an agent where every "harmless"
duplication in the conversation history compounds — the user-history
slice usually gets a fixed token budget, and if 3 of your 10 user
turns are accidental duplicates you're paying 30% of that budget on
noise. The pattern of "dedup contiguous, but reset state at
non-target items" is the right shape for any rolling-window
summarizer input filter; collapsing across non-user items would
erase the temporal structure the summarizer needs to reason about
turn order. The empty-message reset-vs-no-reset decision is the
subtle one — defaulting to "ignore empties for state-tracking
purposes" is usually right because empties are noise, not real
turns.
