# google-gemini/gemini-cli PR #26310 — feat(core): reinforce Inquiry constraints to prevent unauthorized changes

- URL: https://github.com/google-gemini/gemini-cli/pull/26310
- Head SHA: `df3bc43737c243b035a93499ea13e92f2d4e2804`
- Author: akh64bit
- Verdict: **merge-after-nits**

## Summary

Prompt-engineering tweak applied across roughly a dozen `packages/core/src/prompts/snippets.ts` (and adjacent generated/sibling prompt files) all of which share an identical "Expertise & Intent Alignment" bullet. The bullet now (a) gives an explicit example of an Inquiry phrasing ("Can you tell me how to"), (b) extends the Inquiry-mode no-write rule to also cover explicit user requests like "Don't make changes just yet" / "Without changing anything", and (c) replaces "until a corresponding Directive is issued" with "until a subsequent Directive is issued" for clarity.

## Line-level observations

- `packages/core/src/prompts/snippets.ts` line 252 (and parallel lines 2313, 2477, 2635, 2767, 3059, 3481, 3645, 3923, 4087): identical text replacement. Diff is purely additive on each line — no behavior other than prompt content changes.
- The phrase `"Can you tell me how to"` is added as the example after "advice, or observations". This is a reasonable canonical phrasing but it's English-only; in a multilingual product the example will read awkwardly to non-English users. Either drop the example or make it generic ("e.g., questions starting with 'how do I' / 'can you tell me'").
- The new "or whenever the user explicitly instructs you NOT to make changes just yet" clause is good — it pins down the read-only-until-directive contract on a common user phrasing pattern. The two example phrasings ("Don't make changes just yet", "Without changing anything") are reasonable but again English-anchored.
- All the duplicated bullet copies are kept in lockstep — that's the right call (no drift between the variants), but it's also a sign that this file needs a single source-of-truth template constant. The bottom of the diff at line ~252 of snippets.ts (the actual template-literal version with `${options.interactive ? ... }`) is the *real* source; the rest are likely snapshot-test fixtures. If so, this PR is regenerating fixtures correctly.

## Concerns

- This is a pure prompt change, so there's no automated way to validate it doesn't regress instruction-following. A small evaluator pass over a held-out set of "bug observation" prompts (where the previous prompt had agents incorrectly start editing) would meaningfully increase confidence.
- The double-negative in "or whenever the user explicitly instructs you NOT to make changes just yet (e.g., …)" is parseable but tax. Consider rewording to "or whenever the user explicitly tells you to wait before changing anything (e.g., …)".

## Suggestions

1. Reword the new clause to avoid the embedded NOT (suggested above) — easier for LLMs *and* humans.
2. Consider extracting the bullet into a single constant in `snippets.ts` so future edits don't need to touch ~10 sites.
3. Ideally, attach an eval result (pass rate on a "Inquiry vs Directive" classification suite) to the PR description.
