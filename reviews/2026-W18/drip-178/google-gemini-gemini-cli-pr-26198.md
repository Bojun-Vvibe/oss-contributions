# google-gemini/gemini-cli#26198 — fix(core): add explicit empty log guard in A2A pushMessage

- **Repo:** google-gemini/gemini-cli
- **PR:** #26198
- **Head SHA:** dac938561069508dffd895777af57b97ab2c0091
- **Base:** main
- **Author:** community

## Summary

Two-line fix to `A2AResultReassembler.update()`. The previous condition `if (text && this.messageLog[this.messageLog.length - 1] !== text)` evaluates `messageLog[-1]` as `undefined` when the log is empty, then `undefined !== text` is `true`, so by accident it worked for non-empty `text` — *except* the surrounding code path apparently had a separate edge case (issue #24894) where the very first agent message was being dropped. The fix makes the guard explicit: `if (text && (this.messageLog.length === 0 || this.messageLog[last] !== text))`. Adds a 22-line regression test that constructs an `A2AResultReassembler`, calls `update()` with a status-update carrying a single agent message, and asserts `reassembler.toString() === 'First message'`.

## File:line references

- `packages/core/src/agents/a2aUtils.ts:129-133` — guard rewritten to short-circuit on empty `messageLog`
- `packages/core/src/agents/a2aUtils.test.ts:587-607` — new test "should correctly push the first message when messageLog is empty (Issue #24894)"

## Verdict: **merge-after-nits**

## Rationale

Defensive code-clarity fix masquerading as a bug fix — the original `messageLog[length-1] !== text` expression *does* return `true` when length is 0 (since `undefined !== text` for any defined `text`), so on a literal reading of the diff this is a no-op semantically. That means **the PR title and the issue link don't match what the diff actually changes** unless there's a TypeScript `noUncheckedIndexedAccess` or strict-mode wrinkle that made the original expression throw or short-circuit unexpectedly in the production build. Concerns:

1. **The PR body should explain why the original guard didn't catch issue #24894.** As written, the diff is a clarity refactor, not a behavior change. If the actual bug is that `extractPartsText` returns an empty string for the first message in some calling shape (and `text && ...` short-circuits), the fix should be there, not here. Ask the author to paste the failing-before / passing-after invocation that distinguishes the two guards.
2. **The new test at `a2aUtils.test.ts:587-607` passes against both the old AND new guard** (verified by reading the diff: with `messageLog.length === 0`, old expression `[][- 1] !== "First message"` evaluates to `undefined !== "First message"` which is `true`, so `messageLog.push(text)` runs in both versions). Replace with a test that actually distinguishes — e.g., a case where `extractPartsText(parts, '')` returns `""` and the old code did one thing, the new code does another.
3. **Naming: "empty log guard" in the PR title overstates the change.** A better title would be "refactor(core): make A2AResultReassembler empty-log guard explicit" — or, if there *is* a real bug, the PR body needs to demonstrate it.
4. The `as unknown as SendMessageResult` cast at `a2aUtils.test.ts:603` is a code smell. If the `SendMessageResult` type doesn't permit a status-update with a `message` field, the test is exercising a shape the type system says can't happen. Either widen the test type or refine the test fixture to construct a properly-typed payload via a builder.
