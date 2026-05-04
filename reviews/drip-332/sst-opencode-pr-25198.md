# sst/opencode PR #25198 — fix: fix AI refusing to commit

- **Link:** https://github.com/sst/opencode/pull/25198
- **Head SHA:** `dbf6fc674349b1c7e70e8d4862dd1a55631bf188`
- **Author:** scarf005
- **Created:** 2026-05-01
- **Files:** `packages/opencode/src/session/prompt/default.txt` (−2), `packages/opencode/src/session/prompt/trinity.txt` (−2), `packages/opencode/src/tool/bash.txt` (+1/−2)
- **Diff size:** +1 / −6

## Summary
Trims prompt language that the author claims caused the model to over-refuse `git commit` operations. Three prompt files lose two lines each (the `default` and `trinity` system prompts) and `bash.txt` swaps two lines for one.

## Specific citations
- `packages/opencode/src/session/prompt/default.txt` — removes 2 lines (the diff payload; cannot inspect exact text from headers but it is the commit-related instruction block).
- `packages/opencode/src/session/prompt/trinity.txt` — same 2-line removal applied to the trinity persona for consistency.
- `packages/opencode/src/tool/bash.txt` — swaps two lines for one (likely collapsing a "do not run git commit unless asked" instruction).

## Observations
- The change is symmetric across both `default.txt` and `trinity.txt`, which is good — it would be a bug to fix only one persona.
- Prompt-only diffs are notoriously hard to evaluate from the diff alone; behavior should be validated against:
  - a "user explicitly asked to commit" flow → still commits.
  - a "user did not ask, model decided to commit" flow → still refuses (or follows the project's existing safety rule).
- No test changes accompany the prompt edits. There is no regression suite for "the model refused when it should not have committed", which is understandable but means future regressions on this exact symptom will be hard to catch.

## Asks before merging
1. The PR description (or commit body) should quote the *exact* before/after of the removed lines so reviewers can reason about scope without checking out the branch. The 1-line-of-diff-context `git pr diff` output here makes the actual changed text invisible.
2. Confirm with a quick eval that the model still respects the "do not commit unsolicited" guardrail — the rest of the project's safety wording leans on prompt content, so removing two lines could shift behavior in unintended places.

## Verdict
`merge-after-nits`
