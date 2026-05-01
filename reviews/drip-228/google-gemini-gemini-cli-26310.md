# google-gemini/gemini-cli #26310 — feat(core): reinforce Inquiry constraints to prevent unauthorized changes

- **PR**: https://github.com/google-gemini/gemini-cli/pull/26310
- **Head SHA**: `d6ae09bc4e578936c2ceb80c2a1de2b5cf3ee7ba`
- **Files reviewed**: `packages/core/src/prompts/snippets.ts` (+1 -1)
- **Closes**: #24448
- **Date**: 2026-05-01 (drip-228)

## Context

Bug #24448: when a user phrases their request as a clear "don't
change anything yet" Inquiry — e.g. "Don't make changes just yet,
tell me how you'd implement X" or "Without changing anything, can
you explain..." — the model still applied edits and ran tools. The
existing `Expertise & Intent Alignment` system-prompt block already
distinguishes Directives (action requests) from Inquiries (analysis
requests) but only invoked the no-edit constraint on the implicit
"requests are Inquiries unless they contain an explicit instruction"
heuristic. When the user explicitly **negates** the instruction
("don't make changes"), the model was treating the surrounding context
("how do I do X?") as enough of a Directive to override.

This is a one-line prompt-engineering fix that adds the
explicit-negation case to the Inquiry-mode trigger list.

## Diff

`prompts/snippets.ts:255` — single line in the load-bearing
`Expertise & Intent Alignment` bullet:

```diff
-Distinguish between **Directives** (unambiguous requests for action or implementation) and **Inquiries** (requests for analysis, advice, or observations). Assume all requests are Inquiries unless they contain an explicit instruction to perform a task. For Inquiries, your scope is strictly limited to research and analysis;
+Distinguish between **Directives** (unambiguous requests for action or implementation) and **Inquiries** (requests for analysis, advice, or observations). Assume all requests are Inquiries unless they contain an explicit instruction to perform a task. For Inquiries, or whenever the user explicitly instructs you NOT to make changes just yet (e.g., "Don't make changes just yet", "Without changing anything", "Can you tell me how to"), your scope is strictly limited to research and analysis;
```

## Observations

1. **The three example phrases are load-bearing.** Models are trained
   on prompt patterns, and giving three concrete in-quotes example
   triggers (`"Don't make changes just yet"`, `"Without changing
   anything"`, `"Can you tell me how to"`) is the right way to teach
   the model to recognize the negation rather than relying on it to
   abstract from a generic rule. The third example (`"Can you tell
   me how to"`) is the most interesting because it doesn't contain
   an explicit "don't" — it relies on the
   "ask-for-instructions-implies-no-execute" pattern, which is
   exactly the case where the model previously over-applied.

2. **No code-side guardrail.** The fix lives entirely in the system
   prompt; there's no behavioral guardrail in the tool-dispatch path
   that detects negation phrases and refuses to call edit tools.
   That's the right scope for this PR (the bug is "model interprets
   intent wrong", and the fix is "tell the model how to interpret
   intent better"), but it does mean a model that ignores the
   instruction (or runs without the system prompt — e.g. via a
   bypass route) reverts to the old behavior. A future hardening PR
   could add a tool-side detector but that's a much larger change
   with its own false-positive risk.

3. **Phrasing risk.** The expanded bullet now reads as one very long
   sentence with two `For Inquiries...` parallel clauses. The added
   "or whenever the user explicitly instructs you NOT to make changes
   just yet" is a parenthetical inside the larger structure, which
   models generally handle but which makes the rule slightly harder
   for human contributors to read and maintain. A follow-up could
   split this into two bullets ("Inquiries" + "Negation
   constraints") so the contract is more visible.

4. **No test added.** Behavioral system-prompt changes are
   notoriously hard to regression-test, but at minimum a snapshot
   test of `snippets.ts` content would catch accidental reverts.
   The repo presumably has snapshot coverage of the prompt
   already; if not, this is a good moment to add one.

5. **Correct closes-issue reference.** PR body links to #24448 with
   the "Fixes" keyword, so merge auto-closes the bug. Good hygiene.

## Verdict

**merge-as-is** — minimal-surface prompt fix targeting a specific
documented bug, with three well-chosen example trigger phrases that
cover both explicit-negation and ask-for-instructions-implies-
no-execute cases. The two follow-up suggestions (split into
sub-bullets, add snapshot coverage) are gentle improvements, not
blockers.

## What I learned

When the contract you're tightening is "what the model interprets as
permission to act," the load-bearing artifact is example phrases the
model can pattern-match against, not abstract rules. The third
example here (`"Can you tell me how to"`) is the one that does the
real work — it teaches the model to recognize the
ask-for-instructions pattern as Inquiry-mode without relying on
explicit negation. That's exactly the kind of in-quotes example
phrasing that survives prompt revisions and that downstream
contributors can extend by adding more phrases when new escape
patterns are discovered.
