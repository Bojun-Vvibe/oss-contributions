---
pr_number: 19224
repo: openai/codex
head_sha: 205d60adddb5dbc57a84a50e86d43fb14bc77686
verdict: needs-discussion
date: 2026-04-25
---

# openai/codex#19224 — bump phase-two memories model from gpt-5.4 to gpt-5.5

**What changed.** Single line. `codex-rs/core/src/memories/mod.rs` line 70: `pub(super) const MODEL: &str = "gpt-5.4";` → `"gpt-5.5";`. `phase_one::MODEL` and `phase_two::REASONING_EFFORT` are unchanged.

**Why it matters.** Phase-2 memory consolidation is the background job that condenses raw phase-1 memory entries into the canonical store. Model choice here directly shapes long-term agent behavior across resumes.

**Concerns.**
1. **Single-line model bump with the body's only justification being "improved reasoning".** No latency / cost / token-budget data, no consolidation-quality eval. Phase-2 runs on a lease (`LEASE_DURATION_SECS` defined in the same module); a slower model could push lease renewals past the cap and cause two workers to think they own the same job. Worth at least a microbench note.
2. **No corresponding fixture update.** Other recent model bumps in this repo (#19323) updated `models.json` and the related fixtures so default reasoning level and capability flags align. This PR doesn't touch any fixture, which strongly suggests `gpt-5.5` is being used with `phase_two::REASONING_EFFORT = ReasoningEffort::Medium` regardless of whether `gpt-5.5`'s default reasoning surface matches `gpt-5.4`. If the new model has a different default, the unchanged `Medium` may behave differently than `Medium` did on `gpt-5.4`. Confirm or pin explicitly.
3. **Body says "Ran cargo test ... completed successfully".** That's necessary, not sufficient — the entire phase-2 job is exercised against a recorded provider in this codebase only via mocks; a model-name change doesn't break those tests by construction. The reviewer should ask for a one-shot integration run against the live provider, or at minimum a single end-to-end consolidation comparison on a canned phase-1 buffer.
4. **Phase-1 model not bumped.** If consolidation is now richer than the source it consolidates, downstream behavior may regress (consolidator hallucinating structure that phase-1 didn't capture). Either bump both or document why phase-2 is the only one that needs it.

The change itself is trivial; the gap is in the rationale. Hold for a paragraph on why phase-2 specifically benefits and what was measured before/after.
