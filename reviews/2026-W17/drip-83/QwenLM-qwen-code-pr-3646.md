# QwenLM/qwen-code PR #3646 — fix(cli): stabilize sticky todo redraws

- Repo: QwenLM/qwen-code
- PR: #3646
- Head SHA: `8433079e`
- Author: @yiliang114
- Refs: #3638
- Diff: net feature-sized — touches `AppContainer.tsx`, `StickyTodoList.tsx` + tests, `todoSnapshot.ts`, layouts; this is sibling work to #3647 (which merged the panel-compactness side) and #3648.

## What changed

Attacks the streaming-flicker feedback loop between the sticky `Current tasks` panel and Ink's dynamic redraw region. Two interlocking pieces:

1. **Render/layout key split** in `packages/cli/src/ui/utils/todoSnapshot.ts` (new exports `getStickyTodosRenderKey` and `getStickyTodosLayoutKey`). Render key = `id+content+status` (semantic content identity); layout key = `width+maxVisibleItems+id+content` (anything that changes panel height). Status-only updates change render key but not layout key.
2. **`AppContainer.tsx`:1250-1257** introduces `useStableStickyTodos(rawStickyTodos)` — a `useRef`-backed stabilizer that returns the same `todos` reference whenever `getStickyTodosRenderKey` is unchanged, breaking referential-identity churn from `getStickyTodos(historyManager.history, pendingHistoryItems)` running every keystroke. Then `:1594-1632` swaps the `useLayoutEffect` measurement deps from `[shouldShowStickyTodos, stickyTodos]` to `[stickyTodosLayoutKey]`, and gates the `setControlsHeight` calls behind `previousHeight === newHeight ? previousHeight : newHeight` to suppress no-op state updates.

## Specific observations

- The render-key/layout-key split is the right diagnosis. The flicker isn't caused by `TodoWrite` updates per se — it's caused by `setControlsHeight` flipping every status-update tick, which feeds `availableTerminalHeight` and re-runs the dynamic-redraw region. Moving the `useLayoutEffect` deps onto the layout-only key is exactly the fix; `[shouldShowStickyTodos, stickyTodos]` was over-keyed by reference identity. (`AppContainer.tsx:1626-1632`)
- `useStableStickyTodos` (`AppContainer.tsx:170-181`) does the right Ref pattern, but a small concern: `stableTodosRef.current?.renderKey !== renderKey` mutates the ref synchronously during render, which is a common React antipattern (works in concurrent mode only because the ref read is also done in render and `useRef` writes don't trigger re-render). It's safe as written but a `useMemo([renderKey])` would express the same invariant without the in-render ref mutation. Worth a follow-up if maintainers want the cleanest pattern.
- The `setControlsHeight((prev) => prev === h ? prev : h)` guards (`:1607-1620`) are correct and necessary — without them, even with the dep array fixed, the same height value would still trigger React's setState path. Both branches (the `mainControlsRef.current == null` reset and the measured-height write) get the guard, which is right.
- Testing scope: the PR description lists `StickyTodoList.test.tsx`, `todoSnapshot.test.ts`, `AppContainer.test.tsx` etc. but the layout-key behavior — which is the load-bearing change — would be best covered by an `AppContainer.test.tsx` case that asserts `setControlsHeight` is *not* called when only `status` of a todo flips. Worth asking the contributor to add that explicitly so the next refactor doesn't silently regress the dep array back.

## Verdict

`merge-after-nits`

## Rationale

The diagnosis (render-vs-layout key split + dep-array narrowing + setState guards) is correct and addresses the flicker root cause, not just symptoms. Asks: optionally restructure `useStableStickyTodos` to a memo, and add a regression test that asserts no `controlsHeight` write on status-only updates.
