# QwenLM/qwen-code PR #3768 — feat(cli): route foreground subagents through pill+dialog while running

- Head SHA: `e72ff6b9d7f62529b41fd3da8a4de36a02993847`
- Size: +674 / -36, 10 files

## Specific refs

- `packages/core/src/agents/background-tasks.ts:+60/-4` — adds `flavor: 'foreground' | 'background'` discriminator to `BackgroundTaskEntry`. Optional with background-default semantics, so persisted state and older callers behave unchanged.
- `packages/core/src/tools/agent/agent.ts:+66/-5` — synchronous (foreground) Agent path composes a child `AbortController` whose signal is wired to the parent. Cleanup (registry unregister) is in `finally`, covering success / failure / cancel / unexpected throw. Parent-cancel propagates down; child-cancel does not propagate up.
- `packages/cli/src/ui/components/background-view/BackgroundTasksDialog.tsx:507-516` — new `pendingCancelEntryId` state implements two-step cancel for foreground entries. First `x` arms; second `x` confirms. Esc resets without closing the dialog.
- `BackgroundTasksDialog.tsx:617-622` — cancel handler gates on `selectedEntry.flavor === 'foreground' && pendingCancelEntryId !== entryKey`. Background path is one-shot (no behavior change, explicit BC comment).
- `BackgroundTasksDialog.test.tsx:208-263` — three new tests cover foreground two-press confirm, background one-press still works, Esc-resets-armed-cancel without closing dialog. The `pressKey` harness was fixed at `:135-145` to only invoke the latest `useKeypress` handler (previous "call all accumulated handlers" was masking a real-world stale-closure pattern).
- `packages/cli/src/ui/components/messages/ToolMessage.tsx:+70/-11` — branches on `isPending` to render either the inline approval banner (focus-locked agent only) or nothing during the live phase. Committed render path unchanged.

## Assessment

Well-scoped flicker fix with a clear design rationale. The flavor discriminator + optional default = no migration needed is the right call. The `AbortController` composition contract (parent → child only) is documented and tested.

Strong nit on the `pressKey` harness fix: the comment at `:135-141` is gold — explicitly calls out that the old "fire all handlers" was producing a phantom race that doesn't exist in production. That kind of test-hygiene fix usually gets dropped in unrelated PRs; it belongs here because it changes how the new tests behave.

Minor: foreground row prefix `[in turn]` (`:99-103`) is fine but slightly cryptic; consider `[active turn]` or hovering tooltip. Not blocking.

Deferred follow-up (registry not storing live `AgentResultDisplay` reference because `updateDisplay` swaps the object) is correctly called out in the PR body.

verdict: merge-after-nits