---
pr-url: https://github.com/QwenLM/qwen-code/pull/3622
sha: 8b6b0d64f871
verdict: merge-as-is
---

# fix(test): update rewind E2E Test 1 assertion after isRealUserTurn fix

Tiny `scripts/test-rewind-e2e.sh:299-307` change adjusting one E2E assertion after the upstream `isRealUserTurn()` filter started excluding slash commands (e.g. `/rewind`) from the turn-list pre-population. Before this PR, the assertion expected the input bar to contain `"say exactly GAMMA3"` after rewind — which was the previous behaviour where the rewind target was the GAMMA3 user turn (the last real user input) and its text got pre-populated into the input bar. After the upstream filter change, pressing Up once from the initial GAMMA3 selection now lands on BETA2 (because GAMMA3 itself was the last *real* turn, but the navigation skips it and goes to the prior real turn), so BETA2 is the actual rewind target and BETA2's text is what gets pre-populated.

The new assertion `assert_screen "say exactly BETA2"` is correct, and the deletion of the second assertion (`Verify the earlier turns (ALPHA1, BETA2) are still in conversation`) is also correct — because BETA2 is now the *target* of the rewind, it gets removed from the conversation and pre-populated into the input bar, so checking for it in the scrollback would fail. ALPHA1 stays in the scrollback because it's earlier than the rewind target. The new comment block at `:301-305` is unusually thoughtful — it spells out the post-`isRealUserTurn()` navigation contract ("pressing Up once from the initial selection (GAMMA3, the last real user turn) lands on BETA2") which is exactly the kind of "why does this assert what it asserts" annotation that future readers need to avoid "fixing" the test back to the broken pre-PR shape.

The only conceivable nit (not gating): the comment refers to `isRealUserTurn()` by name without a file pointer. A `// see packages/cli/src/.../rewind.ts isRealUserTurn` reference would make the cross-link explicit so the next person updating the rewind navigation knows to re-check this E2E.

## what I learned
When upstream behavior changes, the highest-leverage thing to update is *the test comment*, not the test assertion. The assertion change is mechanical (just match the new output); the comment change is the load-bearing one because it tells future readers *why* the new output is the correct one. Otherwise the test reads like "we assert BETA2 because that's what the script outputs" and the next reader hits "wait, shouldn't this be GAMMA3?" and either flips it back or wastes hours figuring out what `isRealUserTurn` does. The PR author got this exactly right by leading with the navigation-contract paragraph in the comment.
