# block/goose#8995 — feat(chat): group consecutive tool calls into one summarized chain card

- **Head SHA**: `ffb7fc2cbf83`
- **Verdict**: `needs-discussion`

## Summary

Backend support for grouping consecutive `ToolRequest` blocks within a single assistant message into a "chain", and firing one LLM-generated summary covering the whole run once every step has a recorded `ToolResponse`. Adds three new fields to `GooseAcpSession` (`chain_membership`, `responded_tool_ids`, `summarized_chains`), a `ToolChain` struct, an `extract_tool_chains` function, and `with_tool_chain_summary_meta` for emitting the summary as `goose.toolChainSummary = { summary, count }` on the message metadata. +3683/-176 across `crates/goose/src/acp/server.rs` and friends.

## Findings

- `crates/goose/src/acp/server.rs:159-170` — three new HashMaps/Sets per session (`chain_membership: HashMap<String, Arc<ToolChain>>`, `responded_tool_ids: HashSet<String>`, `summarized_chains: HashSet<String>`). `Arc<ToolChain>` is the right call (multiple membership entries point at one chain), but nothing here gets cleared when a session is long-lived and accumulates thousands of tool calls. Need an explicit cleanup story (e.g. drop chain_membership entries for a chain once `summarized_chains.contains(...)` and all responses are accounted for). Otherwise these grow unboundedly per session.
- `crates/goose/src/acp/server.rs:181-186` — `ToolChain.ids` is documented as *"Always `len() >= 2`"*. That invariant lives in a doc comment but isn't enforced at construction time. Either: (a) make `ToolChain::new` fallible, or (b) add a `debug_assert!(ids.len() >= 2)` in the constructor. Otherwise a future caller violating the invariant produces silent single-tool "chains" that fire spurious summaries.
- `crates/goose/src/acp/server.rs:619-651` — `with_tool_chain_summary_meta` does an `unreachable!()` inside a fallback branch that overwrites `goose_entry` if it isn't an Object. The unreachable is reached only if the `*other = ...` assignment changes the variant *and then* the match still doesn't see Object, which is impossible. Fine, but the pattern is hard to read; a helper like `meta.get_or_insert_object("goose")` would be clearer and avoid the unreachable.
- `crates/goose/src/acp/server.rs:701-720` — `extract_tool_chains` deliberately treats `MessageContent::ToolResponse` as chain-neutral with the comment *"so a stray response doesn't split a chain if the data shape ever changes"*. That's defensive, but it means if the shape *does* change and assistant messages start carrying responses, this code will silently misgroup. Add a `tracing::warn!` in that arm so the next contributor sees the rationale fail loudly rather than quietly.
- General: this is a 4558-line diff touching ACP session state machinery. The chain summary fires a separate LLM call per chain; that's a meaningful cost increase per turn for tool-heavy agents. The PR description should quantify (or at least flag) the additional token spend, and ideally the summary call should be gated by config — some users won't want extra LLM round-trips for every multi-tool step.
- The chain-completion check is keyed on "all responses present" but doesn't appear (in the visible portion of the diff) to handle the case where a tool *errors* and never produces a response. Need confirmation that `handle_tool_response` is called even on tool failure, or `summarized_chains` will never gate-open for chains containing a failed tool.
- Frontend chain detection is referenced (`MessageBubble.groupContentSections`) at `:697`. Backend chain semantics need to stay in lockstep with the frontend; an integration test or shared fixture spec would catch drift early.

## Recommendation

The feature is well-scoped and the data structures are reasonable, but: (1) unbounded per-session memory growth, (2) silent misgrouping on shape change, (3) unmeasured extra LLM cost, and (4) ambiguous behavior on tool errors are all worth a design pass before merging. Recommend a follow-up discussion thread before this lands — particularly the cost-gating decision, since it changes the per-turn LLM bill for every multi-tool agent.
