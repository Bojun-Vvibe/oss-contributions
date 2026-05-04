# QwenLM/qwen-code #3836 — feat(core,cli): surface and cancel auto-memory dream tasks

- URL: https://github.com/QwenLM/qwen-code/pull/3836
- Head SHA: `3d8b978bafee255f15798c690ed083d2ac73c07d`
- Author: wenshao
- Size: +947 / -20 across 13 files (background-view UI + memory dream wiring)

## Comments

1. `packages/cli/src/ui/components/background-view/BackgroundTasksDialog.test.tsx:40-44` — New `case 'dream': return entry.dreamId;` extends the registry-key switch correctly. Make sure the production code's exhaustiveness check (`_exhaustive: never`) is also updated in the same file to actually cover `dream`, otherwise `kind: 'dream'` reaches the `default` branch in non-test code.
2. `BackgroundTasksDialog.test.tsx:74-86` (`dreamEntry`) — Helper builds a `DreamDialogEntry` literal — good defaults. Consider adding a `metadata.touchedTopics: []` default so each test doesn't have to override individually.
3. `BackgroundTasksDialog.test.tsx:540-555` (`routes 'x' on a running dream...`) — Strong test: asserts both the positive (`cancelTask('d-zzz')`) and the negative (`cancel`/`monitorCancel` not called). This is exactly the kind of guard that prevents the wrong AbortController from being signalled. Keep it.
4. `BackgroundTasksDialog.test.tsx:584-598` (`omits the topics block entirely while...running`) — Pinning that the section header is hidden (not just empty) is correct UX. Verify the production component returns `null` rather than an empty `<Section>` to keep DOM/ink-tree clean.
5. `packages/core/src/memory/dream.ts` and `dreamAgentPlanner.ts` — Diff window truncated; please confirm the `cancelTask` path actually aborts in-flight LLM streams (the planner can hold an open SSE that will keep tokens flowing post-cancel).
6. `packages/core/src/tools/task-stop.ts` — Make sure `task_stop` validates the `dreamId` against the live MemoryManager registry to avoid no-op cancels silently confusing the user.
7. PR description should clarify: this looks like PR-2 of a stack (the test comment references "PR-1 read-only surface"); link the predecessor in the description so reviewers can read them in order.

## Verdict

`merge-after-nits`

## Reasoning

This is a substantial but well-tested vertical slice: the dream-task lifecycle is wired through the unified background-view pill/dialog, the MemoryManager gains a `cancelTask` hook, and the TUI gets an `x stop` affordance. The test file is unusually high-quality — explicit "pin the dream-cancel branch" comments document why each assertion exists. Main asks: ensure the producer-side `cancelTask` actually aborts the LLM stream, and verify the exhaustive-switch update in the dialog component (not just the mock). Otherwise the contract is sound.
