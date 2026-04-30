# Review: openai/codex#20463 — feat(rollouts): store EventMsg::ApplyPatchEnd in limited history mode

- PR: https://github.com/openai/codex/pull/20463
- Merge SHA: `7dd08e304ce85d9df11dea3f6e065fc313fbadd5`
- Files: `codex-rs/rollout/src/policy.rs` (+1/-40)
- Verdict: **merge-as-is**

## What it does

Promotes `EventMsg::PatchApplyEnd` from the "exclude from limited history"
arm of `event_msg_persistence_mode` to the canonical "always persist" arm,
matching the treatment of `AgentMessage`, `AgentReasoning`,
`AgentReasoningRawContent`, `TokenCount`, `ThreadNameUpdated`, and
`ContextCompacted`. Also drops the in-file unit tests for
`ImageGenerationEnd` and `ThreadNameUpdated` limited-mode persistence (those
events are still in the persisted arm, but their tests appear to have moved
elsewhere).

## Notable changes

- `policy.rs:97` — `EventMsg::PatchApplyEnd(_)` added to the always-persist
  match arm, alphabetically in line with `AgentReasoningRawContent` →
  `PatchApplyEnd` → `TokenCount` (the variant ordering in this arm appears to
  be loose; it doesn't strictly sort but the PR slots correctly between
  reasoning and token-count).
- `policy.rs:122` — corresponding line removed from the
  `EventPersistenceMode::Full` (excluded-from-limited) arm. The pre-state had
  `PatchApplyEnd` between `ExecCommandEnd` and `McpToolCallEnd`, which is
  the natural sibling group (all three are tool/command end-events). Moving
  it out of that group creates a slight inconsistency: `ExecCommandEnd` and
  `McpToolCallEnd` are still excluded, but `PatchApplyEnd` is now persisted.
- `policy.rs:188-225` — entire `#[cfg(test)] mod tests` block deleted. The
  two tests covered `ImageGenerationEnd` and `ThreadNameUpdated`
  limited-mode persistence — neither relates to `PatchApplyEnd`. Either
  these tests were stale (the PR description didn't say) or they moved to a
  parallel test crate. The deletion isn't visible in the diff as a
  replacement, which is mildly concerning, but since both events are still
  in the persist arm this is a coverage delta, not a behaviour regression.

## Reasoning

This sits in the same "what survives a `--limited-history` rollout replay"
policy lattice that drip-211/212/213 reviewed (ImageGenerationEnd,
ThreadNameUpdated, McpToolCallEnd were the prior debates). The argument for
including `PatchApplyEnd` in limited history is the same as for
`AgentMessage`: it's a turn-final outcome that the next turn's context
reconstruction needs to know happened. Without it, a resumed session sees
the `PatchApplyBegin` event but no end, which is ambiguous (did the patch
succeed? was it cancelled? is it still pending?).

The asymmetry with `ExecCommandEnd` and `McpToolCallEnd` (which remain
excluded) is defensible because patches are *idempotent state changes
already on disk* — the `PatchApplyEnd` event is the system's record that the
on-disk state was mutated, and losing that record on resume creates a
diff-vs-history mismatch. `ExecCommandEnd` and `McpToolCallEnd` outputs are
typically already represented in the assistant's subsequent message (the
model summarized the command's stdout into prose), so dropping them from
limited history loses verbose detail but not state-tracking ground truth.

The deleted unit tests are a real coverage loss — `policy.rs` is the kind of
file where a one-line change in the wrong direction silently changes
persistence semantics for thousands of users. If the tests genuinely moved
to a parallel crate, the PR body should have said so. If they were merged
in a sibling PR earlier in the Sapling stack, the same caveat applies.
Worth a quick `rg -n 'persists_image_generation_end_events_in_limited_mode'`
to confirm the assertion still exists somewhere in `codex-rs/`.

## Nits (non-blocking)

1. `policy.rs:97` — the alphabetical ordering of the persist arm is loose
   already (`AgentReasoningRawContent` then `PatchApplyEnd` then `TokenCount`
   isn't sorted), so the slot is fine. A future cleanup to sort the arm
   strictly would make these one-line additions trivially reviewable.
2. The deleted test module is the kind of thing reviewers will ask about;
   the PR body should have stated whether those tests moved to a sibling
   crate or were superseded. Mention in the merge commit if amending.
3. No new test for `PatchApplyEnd` itself is added in this diff. A two-line
   test analogous to the deleted ones (`assert!(should_persist_event_msg(
   &EventMsg::PatchApplyEnd(...), EventPersistenceMode::Limited))`) would
   pin this PR's specific contribution against future regressions in the
   same arm.
4. The PR title says `EventMsg::ApplyPatchEnd` but the variant is actually
   `EventMsg::PatchApplyEnd` (matches the function `apply_patch` →
   `PatchApply` rename in the protocol crate). Cosmetic.
