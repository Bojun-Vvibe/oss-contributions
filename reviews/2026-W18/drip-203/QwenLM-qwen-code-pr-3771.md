# QwenLM/qwen-code #3771 — fix(cli): restore SubAgent shortcut focus

- **Author:** yiliang114
- **SHA:** `a776194`
- **State:** OPEN
- **Size:** +220 / -3 across 2 files
  (`packages/cli/src/ui/components/messages/ToolGroupMessage.tsx`,
  `packages/cli/src/ui/components/messages/ToolGroupMessage.test.tsx`)
- **Verdict:** `merge-after-nits`

## Summary

Restores Ctrl+E / Ctrl+F shortcut focus for normally-running subagent
displays. In v0.15.6 the shortcut listener depended on `isFocused`, but that
signal originally tracked the subagent confirmation focus lock — so a
running subagent could display the "+N more (ctrl+e to expand)" hint while
the shortcut listener stayed inactive. Fix at `ToolGroupMessage.tsx:33-43`
adds an `isRunningAgent` type predicate, then at `:143-153` derives a new
`runningSubagentCallId` (the first running subagent in array order) via
`useMemo`, and combines it with the existing `focusedSubagentCallId` into a
single `keyboardFocusedSubagentCallId = focusedSubagentCallId ??
runningSubagentCallId` at `:154-156`. The render branch at `:280-292` then
gates `isSubagentFocused` on this combined id with explicit priority
("Pending confirmation takes strict priority over running fallback"). Locked
by 4 new test cases in the new `SubAgent focus` describe block at
`ToolGroupMessage.test.tsx:278-454` covering single-running, first-of-many,
pending-wins-over-running, and tool-confirmation-blocks-all.

## Reasoning

The diagnosis is correct and the fix shape is correct:

1. **Pre-PR semantics conflated two distinct focus signals.** "Has any
   subagent claimed the inline confirmation focus lock?" is a different
   question from "Should Ctrl+E expand the first running subagent?". The
   old code answered both with the same boolean
   (`focusedSubagentCallId !== null`), so when no subagent was awaiting
   confirmation the keyboard shortcut was always inactive even when the
   compact hint was visible. Splitting the two via `??` (pending wins) is
   the minimum-blast-radius fix.

2. **`useMemo` with `[toolCalls]` dep is the right reactivity shape.**
   `runningSubagentCallId` is recomputed only when the tool-calls array
   identity changes, which matches how the parent re-renders this group
   on subagent state transitions. `find(...)?.callId ?? null` is cheap
   enough that the memo is more about preventing useless prop diffs to
   the children than about the find cost itself.

3. **The 4 new tests cover the priority lattice exhaustively.** The
   "pending-wins-over-running" case at `:344-393` and the
   "tool-confirmation-blocks-all" case at `:401-453` are the load-bearing
   ones — they lock the two precedence rules that the old code couldn't
   express. The mock at `ToolGroupMessage.test.tsx:30-51` correctly
   discriminates `task_execution`-typed `resultDisplay` from generic
   tools and forwards `isFocused` so the assertion is on the actual prop
   the renderer receives, not on internal state.

Two nits worth a follow-up before merge:

1. **"First in array order is the oldest" assumption deserves a defensive
   assertion.** The comment at `:142-145` correctly explains the rationale
   ("oldest — most likely to have accumulated tool calls and display the
   '+N more' hint") but the invariant ("toolCalls preserves insertion
   order") is implicit in the parent. A future refactor that sorts
   toolCalls by status or recency would silently break the focus heuristic
   without breaking any test (because every test in the new `SubAgent
   focus` describe block constructs its own ordered array). A one-line
   sanity test like "moves focus when first running subagent transitions
   to completed" would lock the property.

2. **The `isFocused && !toolAwaitingApproval &&
   keyboardFocusedSubagentCallId === tool.callId` predicate at `:285-288`
   is now the third triple-`&&` site in this file** (the other two are
   the existing confirmation-lock and waiting-indicator gates). Extracting
   a small `shouldOwnSubagentShortcuts(tool, ctx)` helper would make the
   precedence rules grep-able for the next reader; today understanding
   "why doesn't Ctrl+E work right now" requires reading three separate
   ternaries against three slightly different contexts.

The fix is correct, the tests are dense and exercise the right invariants,
and the `?? `-as-fallback shape is a clean expression of "pending strict
priority". Worth merging once the order-dependent regression test lands.
