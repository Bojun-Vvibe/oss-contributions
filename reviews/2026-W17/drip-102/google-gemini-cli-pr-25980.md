# google-gemini/gemini-cli PR #25980 — fix(cli): don't crash when an @-mention captures a non-path blob

- Link: https://github.com/google-gemini/gemini-cli/pull/25980
- Head SHA: `01bdd32b7f472891cec310ed9a1431a1417107a7`
- Size: +70 / -3 across 2 files

## Summary

Wraps the `resolveToRealPath()` call inside `checkPermissions()` (`packages/cli/src/ui/hooks/atCommandProcessor.ts:188-208`) in a `try/catch` so that an `@`-token whose body is actually a pasted JSON/code blob (which the greedy `@`-regex captured up to the next ASCII delimiter) no longer propagates `ENAMETOOLONG`/`EINVAL` from `fs.realpathSync` as an unhandled rejection that kills the interactive session. Closes #22029, related to #25910 / #25923.

## Specific-line citations

- `atCommandProcessor.ts:188-208`: the `let resolvedPathName; try { resolvedPathName = resolveToRealPath(...) } catch { continue }` shape, with a comment explicitly naming the greedy-regex root cause and the "permission gating is a pre-flight check" rationale for treating "can't resolve" as "skip the entry".
- `atCommandProcessor.test.ts:1542-1597`: two new tests under a new `describe('checkPermissions')` block — one drives a single `@` followed by `'a'.repeat(8192)` and asserts `resolves.toEqual([])`, the second drives `@real.txt and @${'b'.repeat(8192)}` and asserts the real file is still resolved, which is the right shape because it pins both the no-crash invariant and the "don't drop sibling real mentions" non-regression.
- The PR body explicitly leaves `paths.ts:robustRealpath` strict so other internal callers (resource registry, agent definitions) keep their fail-loud semantics — only the user-input boundary is loosened.

## Verdict

**merge-as-is**

## Rationale

This is a textbook input-validation fix at the right layer: the bug only exists at the user-input → filesystem boundary, the fix is isolated to that boundary, the comment captures the future-reader's "why is there a swallow here" question, and the regression tests pin both directions. The greedy-regex itself is not touched, which is correct — fixing the regex would risk changing `@`-completion semantics for legitimate paths-with-spaces edge cases, while the catch is observably safe (the only thing thrown away is an entry that was never going to resolve anyway).

The only thing I'd add as a follow-up (not blocking) is a `console.debug` on the catch path so the underlying greedy-regex / hallucinated-path-arg bug stays visible to anyone who grep-debugs why an `@`-mention silently dropped — but that's an enhancement, not a merge condition.
