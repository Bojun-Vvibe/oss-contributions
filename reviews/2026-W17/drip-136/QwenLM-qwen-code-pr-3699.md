# QwenLM/qwen-code #3699 — fix(core): treat ask_user_question multiSelect as optional

- PR: https://github.com/QwenLM/qwen-code/pull/3699
- Head SHA: `99f8e0b85062`
- Diff: 44+/4- across 3 files (`packages/core/src/tools/askUserQuestion.test.ts`, `packages/core/src/tools/askUserQuestion.ts`, `packages/core/src/tools/tools.ts`)
- Base: `main`

## Verdict: merge-as-is

## Rationale

- **Closes a real "model omits a boolean and gets a hard validation rejection" footgun.** Today, `multiSelect: boolean` is in the `required` array of the function declaration at `askUserQuestion.ts:116`, and the validator at `:348` does an unconditional `typeof question.multiSelect !== 'boolean'` check. Models that emit a perfectly-valid single-select question (the common case — `multiSelect` defaults to false semantically) without the field get rejected with `Question N: "multiSelect" must be a boolean.` The PR fixes all three layers (interface signature, function-declaration `required` list, validator gate) so the field is genuinely optional.
- **The validator change is the load-bearing one.** `:348-351` switches from `if (typeof question.multiSelect !== 'boolean')` to `if (question.multiSelect !== undefined && typeof question.multiSelect !== 'boolean')` — the new gate accepts both omitted (`undefined`) and any explicit boolean value, but still rejects `multiSelect: "yes"` / `multiSelect: 1` / etc. That's exactly the right contract for an optional typed boolean.
- **Removing `'multiSelect'` from `required` at `:116`** propagates correctly into the JSON Schema sent to the model, so the model itself sees the field is optional and won't be confused into emitting a placeholder false value that gets logged as "user explicitly chose single-select" when the model meant "I didn't think about it".
- **Interface signature change at `askUserQuestion.ts:36` (`multiSelect: boolean` → `multiSelect?: boolean`) and the matching change in `tools.ts:700` (`ToolAskUserQuestionConfirmationDetails`) keep the TypeScript type system aligned with the runtime contract.** Without both, downstream `if (q.multiSelect)` checks would still type-narrow the wrong way.
- **Test coverage is precise and complete.** `askUserQuestion.test.ts:101-118` covers the positive case (omitted `multiSelect` validates and builds without throwing); `:120-138` covers the negative case (`multiSelect: 'yes'` produces the exact error string `'Question 1: "multiSelect" must be a boolean.'`). That two-cell pair locks the contract: optional, but if present must be the correct type. The `as unknown as boolean` cast at `:135` is the right idiom for testing a runtime validation that the static types would otherwise prevent.

## Nits / follow-ups

- **No test for explicit `multiSelect: true` and `multiSelect: false` after the change.** The existing tests presumably cover this via earlier suites, but a one-line confirmation in this PR's test file that explicit booleans still pass would defend against accidental future tightening.
- **Consider whether the runtime *consumer* of `multiSelect` defaults correctly.** The PR makes the field optional in input validation but doesn't show what happens at the rendering / interaction layer when `multiSelect === undefined`. The expected behavior is "treat as `false` (single-select)" but a default at the consumption site (`question.multiSelect ?? false`) would be worth verifying. The diff doesn't show that downstream change, so worth a follow-up grep for `\.multiSelect` consumers in the codebase.
- **`required` list reduction is a soft API change for any external integrators consuming the function-declaration JSON.** If anyone has built tooling that introspects the `required` array, they'll see a change. Probably non-load-bearing but worth a CHANGELOG line.

## What I learned

This is a textbook "the schema said `required` but the field always had a sensible default" mismatch. The fix is small but the lesson is that JSON-Schema `required` should reflect "the caller must commit to a choice" — not "we'd like to see this populated". When the default is unambiguous (omitted ≡ false here), pulling the field out of `required` lets the model emit cleaner output and removes a class of validation-error retries. The two-cell test (positive omit + negative wrong-type) is the right shape for any "make this field optional" change — the negative case is what prevents the change from accidentally widening to "any value goes".
