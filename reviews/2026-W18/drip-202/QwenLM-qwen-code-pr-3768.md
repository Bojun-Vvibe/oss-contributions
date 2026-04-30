# QwenLM/qwen-code #3768 — feat(cli): route foreground subagents through pill+dialog while running

- **Author:** tanzhenxin
- **SHA:** `e72ff6b`
- **State:** OPEN
- **Size:** +674 / -36 across `BackgroundTasksDialog.{tsx,test.tsx}`, `HistoryItemDisplay.tsx`, `messages/{ToolGroupMessage,ToolMessage,ToolMessage.test}.tsx`, `subagents/*.tsx`, agent runtime
- **Verdict:** `merge-after-nits`

## Summary

Suppresses the live inline `AgentExecutionDisplay` for foreground (synchronous)
agent tool calls and routes them through the existing footer pill +
`BackgroundTasksDialog` instead, eliminating a flicker class where
`pendingHistoryItems` repaints during long-running verbose subagents produced
visible jittery scrolling. Three load-bearing pieces:

1. **A new `flavor: 'foreground' | 'background'` discriminator on
   `BackgroundTaskEntry`** (optional, defaults to background semantics) gates
   `emitNotification`, `hasUnfinalizedTasks`, and the cancel grace timer —
   foreground entries bypass all three.

2. **A new `isPending` prop threaded through `HistoryItemDisplay →
   ToolGroupMessage → ToolMessage → SubagentExecutionRenderer`** at
   `HistoryItemDisplay.tsx:213`, `ToolGroupMessage.tsx:39-46,69,304`. Live
   render branch decides "approval banner" vs "nothing"; committed render
   path is unchanged so scrollback view is identical post-commit.

3. **A two-step cancel in `BackgroundTasksDialog.tsx:507-516,608-624,634-682`**
   — foreground entries require two `x` presses (first arms a
   `pendingCancelEntryId` state, second confirms; Esc resets) so a stray
   key can't end the parent's current turn. Background entries keep the
   one-shot one-press cancel UX. Hint footer at `:692-704` exposes the
   armed state with "x again to confirm stop · ends current turn".

Test coverage at `BackgroundTasksDialog.test.tsx:208-263` locks all three
shapes — foreground requires two presses, background still one-press, Esc
backs out without closing the dialog.

## Reasoning

The redesign is the right call — `pendingHistoryItems` repaint is a
fundamental Ink limitation for tall/mutating frames, and routing through a
fixed-position pill removes the flicker class entirely without touching
scrollback. The two-step cancel UX is the right safety gate (cancelling a
foreground subagent ends the parent's turn with a partial result —
genuinely destructive). The `flavor` discriminator being optional with
default-background semantics is the right backwards-compat shape. Specific
quality wins:

1. **The test harness fix at `BackgroundTasksDialog.test.tsx:135-145`**
   ("only fire the most recently registered handler") is a bug fix in the
   test infrastructure, called out with a comment naming the production
   semantic ("Real `useKeypress` unbinds the previous callback on rerender,
   so only the most recently registered closure should run. Calling all
   accumulated handlers misses state updates that happened between renders
   — the symptom looks like a re-render race in production code that
   doesn't exist."). That comment will save someone a half-day debug session
   later.

2. **The `[in turn]` row prefix at `BackgroundTasksDialog.tsx:99-111`** is a
   real affordance — users don't have to memorize the cancel-consequence
   table.

Three pre-merge concerns:

1. **The PR body acknowledges "a queued approval (when two parallel
   subagents both ask for approval) is no longer visible in the main view —
   only in the dialog — until the focus lock rotates to it."** That's a
   real UX regression for power users running parallel subagents. A
   visible-in-pill counter ("2 agents · 1 awaiting approval") would
   surface the queue without re-introducing the inline-frame flicker.

2. **`isPending` is wired through four component layers but is `optional`
   with default `false`.** A consumer that forgets to forward it (or a
   future refactor that adds a fifth layer between `ToolMessage` and
   `SubagentExecutionRenderer`) will silently re-introduce the inline
   live-frame for foreground subagents and the flicker class returns. A
   compile-time required prop at the renderer leaf would lock this
   structurally — the cost is one explicit `isPending={false}` at the
   committed-render call site.

3. **Cleanup is in `finally` so we always unregister on success, failure,
   cancel, or unexpected throw"** — confirm the composed `AbortController`
   in `agent.ts`'s synchronous path tears down the registry entry on the
   "parent abort propagates down, child cancel does not propagate up"
   asymmetric path. A regression test for "parent turn aborted while
   foreground subagent in flight → no orphan pill entry" would lock the
   property.
