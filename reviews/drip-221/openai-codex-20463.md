---
pr-url: https://github.com/openai/codex/pull/20463
sha: 7dd08e304ce8
verdict: merge-after-nits
---

# feat(rollouts): store EventMsg::ApplyPatchEnd in limited history mode

Two-line semantic move at `codex-rs/rollout/src/policy.rs`: `EventMsg::PatchApplyEnd(_)` migrates from the "skip in limited mode" arm at `:122` (deleted) into the "always persist" arm at `:97` (added between `AgentReasoningRawContent` and `TokenCount`). Same begin/end-asymmetry pattern as drip-219's #20464 (which moved `McpToolCallEnd` for the MCP tool-call surface): limited-history rollouts had been persisting `PatchApplyBegin` while dropping `PatchApplyEnd`, leaving resumed sessions in the "patch in flight" state when the patch had in fact already been applied — the most-pernicious half-state for replay because the next turn's reasoning sees an *unresolved* file-modification operation and may decide to re-issue, re-stat, or re-apply the patch.

The fix is the right shape and follows the precedent set by #20464 verbatim — same arm, same insertion ordering style, paired with deletion of the `#[cfg(test)] mod tests` block at `:188-223`. That test-deletion is the only nit worth raising: the deleted tests asserted `should_persist_event_msg(&ImageGenerationEnd, Limited) == true` and `should_persist_event_msg(&ThreadNameUpdated, Limited) == true`, which are exactly the kind of "this variant is in the always-persist arm" assertions that should grow when *new* variants get moved into the always-persist arm, not shrink when they do. The defensible argument for deletion is "these tests asserted variants that haven't moved, they're not load-bearing for this PR's contract" — but the *cumulative* loss of coverage is real: after #20464 + this PR, the always-persist arm now has `McpToolCallEnd`, `PatchApplyEnd`, `ImageGenerationEnd`, `ThreadNameUpdated`, `ApplyPatchEnd` (5 begin/end pairs and lifecycle events), zero of which have a "this is in the always-persist arm" pin. A `parametrize`-style test enumerating the always-persist set against a hardcoded list would be the right shape — the next reviewer can no longer answer "did you mean to remove that variant from always-persist" by looking at test diff.

The other nit (smaller): the PR description should have explicitly named the prior #20464 as the "same shape, sibling EventMsg" precedent so future bisecters tracking down "when did limited-mode rollout-replay correctness get fixed for the patch-apply path" can find both PRs in one search rather than discovering #20464 by reading `policy.rs` history three months later.

## what I learned
Begin/end-event asymmetry in persisted state is the worst possible replay-correctness state — strictly worse than dropping both events (replay sees nothing happened, may re-issue the same operation but at least starts from a clean state) because dropping only the `End` half teaches replay that the operation is mid-flight, which triggers retry logic that double-applies. Every persistence-policy review for an event stream should ask "for every Begin variant in the persist arm, is the matching End variant in the same arm?" as a standing checklist item.
