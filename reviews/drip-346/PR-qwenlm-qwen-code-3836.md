# QwenLM/qwen-code#3836 — feat(core,cli): surface and cancel auto-memory dream tasks

- PR ref: `QwenLM/qwen-code#3836` (https://github.com/QwenLM/qwen-code/pull/3836)
- Head SHA: `3d8b978bafee255f15798c690ed083d2ac73c07d`
- Title: feat(core,cli): surface and cancel auto-memory dream tasks
- Verdict: **needs-discussion**

## Review

The user-visible motivation is solid: auto-memory consolidation ("dream") tasks
today fire silently from `MemoryManager.scheduleDream` and the user has no way
to know what's running, what it's reviewing, or how to stop a runaway
consolidation that's burning tokens. Surfacing them in the unified
BackgroundTasks pill / dialog and adding a cancel path is the right product
direction.

The dialog plumbing is well structured. The `DreamDialogEntry` discriminated
union extension (`packages/cli/src/ui/hooks/useBackgroundTaskView.js` +
`BackgroundTasksDialog.test.tsx` lines 18-21, 57-77 of the diff) follows the
existing pattern set by `AgentDialogEntry` and `MonitorDialogEntry` cleanly,
and the exhaustiveness check (`const _exhaustive: never = entry`) at line ~42
of the test file's mock means a new task kind in the future will fail to
typecheck instead of silently falling through.

What I'd want to discuss before approving:

1. **Cancellation surface ambiguity.** The PR adds three different ways to
   cancel a dream: dialog `x` keystroke, `task_stop` tool dispatch, and
   `MemoryManager.cancelTask`. The PR body itself notes that "cancellation
   aborts the dream fork-agent and lets the existing `runDream` finally
   block release the consolidation lock as the agent unwinds" — that's
   load-bearing behavior that depends on `runDream`'s finally block running
   to completion under abort. Worth a test that explicitly asserts the
   consolidation lock is released after a cancel, otherwise a future
   refactor of `runDream` could silently leak the lock and dreams would
   stop firing entirely.

2. **`task_stop` dispatch route.** Adding "a 4th dispatch route so the model
   can also cancel" gives the LLM a new ability to terminate a background
   task it didn't start. That's a reasonable agent capability but it
   warrants a one-paragraph justification in the PR body about why the
   model should be allowed to cancel a user-initiated dream consolidation
   it has no context for. Default-deny would be safer; if the model tries
   to cancel and it's denied, the user can still cancel via the dialog.

3. **Scope.** 947 additions across cli + core touching dialog state, tool
   dispatch, MemoryManager status enum, and fork-agent abort wiring is a
   lot for one PR. If the per-kind detail body / pill-count work is
   independent of the cancel-via-tool work, splitting would make this much
   easier to review and revert if needed.

Read-only surface (the new `[dream] memory consolidation reviewing N
sessions` row, the detail body) is uncontroversial. Cancel surface needs
the discussion above.
