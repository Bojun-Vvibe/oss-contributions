# Review: QwenLM/qwen-code PR #3771 — fix(cli): restore SubAgent shortcut focus

- **Head SHA:** `d29cbceb63a2db16ee339b3a28ca30b99e216550`
- **State:** MERGED (2026-04-30)
- **Size:** +247 / -5, 4 files
- **Verdict:** `merge-as-is`

## Summary
Fixes a regression introduced in v0.15.6 where a normally-running SubAgent
display would advertise `ctrl+e to expand` in its compact tool-call summary
but the Ctrl+E / Ctrl+F shortcut listener stayed inactive. The shortcut
listener gates on `isFocused`, which historically tracked SubAgent
*confirmation* focus only. The fix assigns shortcut ownership to the first
running SubAgent when no pending confirmation owns it.

## What's good
- The focus priority rule encoded by the diff
  (`packages/cli/src/ui/components/messages/ToolGroupMessage.tsx:+30/-3`)
  is exactly the documented hierarchy:
    1. Direct tool-level confirmation blocks all SubAgent shortcut focus.
    2. Pending SubAgent confirmation wins next.
    3. Otherwise, the first running SubAgent owns the shortcut.
  This matches the reviewer focus checklist in the PR body.
- The test additions at
  `packages/cli/src/ui/components/messages/ToolGroupMessage.test.tsx:+209/-0`
  are unusually thorough for a UX fix — five distinct scenarios:
  normal-running, parent-not-focused, multiple-running (only-first-wins),
  pending-confirmation-wins-over-running, and direct-tool-confirmation-blocks-all.
- Mock harness at `ToolGroupMessage.test.tsx:27-47` is properly typed (no
  `any` slips in via `resultDisplay`/`isFocused`) and the mock branches on
  `task_execution` so the SubAgent path renders independently from the
  normal tool path.
- Touches three non-test files
  (`ToolGroupMessage.tsx`, `ToolMessage.tsx:+4/-1`,
  `AgentExecutionDisplay.tsx:+4/-1`) and the changes are minimal — propagate
  `isFocused` through the result-display chain to where the shortcut listener
  reads it.

## Minor observations (not blocking — already merged)

1. **First-running rule is order-dependent.** When multiple SubAgents are
   running in parallel, "first" is positional in the `toolCalls` array. The
   PR body acknowledges this as a tradeoff "until there is a fuller SubAgent
   selection UX." Worth tracking as a follow-up issue if not already.

2. **No real TUI validation video** (called out by author in the testing
   matrix). Vitest + typecheck + Prettier covered the focus state machine
   but not the actual `ctrl+e` keybinding interaction with the new focus
   owner. Possible follow-up: add a Pact-style key-event integration test.

3. The `pendingDisplay` test at
   `ToolGroupMessage.test.tsx:392-417` uses `vi.fn()` for `onConfirm` —
   ensure subsequent tests don't share the spy, or add `vi.restoreAllMocks()`
   to the test's `beforeEach`. Minor hygiene.

4. Windows / Linux columns in the testing matrix are flagged as ⚠️ — the
   focus state machine is platform-agnostic, but Ctrl+E on Windows in some
   terminal emulators emits different sequences. Worth a smoke check.

## Verdict rationale
Already merged, and rightly so. The fix is surgical, the focus priority is
correct, the test coverage exceeds what's typical for a TUI focus regression,
and the linked issue (#3763) is closed by it. Nothing in the diff suggests
the change can leak focus into completed/history SubAgents (the running-state
guard at `ToolGroupMessage.tsx` is explicit about it).
