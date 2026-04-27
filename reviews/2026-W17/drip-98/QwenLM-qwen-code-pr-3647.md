# QwenLM/qwen-code#3647 — fix(cli): keep sticky todo panel compact

- **Head**: `16753ebda5dac9fb17f57e1d5ffa0d3303113573`
- **Size**: +463/-37 across 8 files
- **Verdict**: `merge-after-nits`

## Context

Fixes #3638 (Windows PowerShell flicker on long todo lists) and addresses #3507 follow-up (long lists eating vertical space + duplicate rendering when the source `TodoWrite` tool result is still on screen). Three orthogonal changes bundled into one PR:

1. **Compact rendering**: limit visible sticky todo items based on terminal height; render rows as single-line truncated; show overflow as `… and N more`.
2. **Visibility guard**: hide the sticky panel while an inline `TodoWrite` result is pending and for a short window after the committed `TodoWrite` history item is still likely visible.
3. **Render-stability fix** (the flicker): stable layout key + memoized footer-height measurement so status-only todo updates don't trigger a footer remeasure every time a status flips.

## Design analysis

The flicker fix at `packages/cli/src/ui/AppContainer.tsx:166-186, 1592-1626` is the most subtle and most important piece. The previous code had `useLayoutEffect` deps that included `shouldShowStickyTodos` and `stickyTodos` directly — both change shape constantly during streaming (`stickyTodos` is a fresh array every render), so `measureElement(mainControlsRef.current)` ran on every status flip, calling `setControlsHeight` with the same value, which Ink translated into a full bottom-area reflow → PowerShell flicker.

The fix replaces those deps with a single `stickyTodosLayoutKey` derived via `getStickyTodosLayoutKey(stickyTodos, width, maxVisible)` — a string key that changes only on layout-affecting fields. Combined with `useStableStickyTodos` at `:166-180`:

```ts
function useStableStickyTodos(todos: TodoItem[] | null): TodoItem[] | null {
  const renderKey = getStickyTodosRenderKey(todos);
  const stableTodosRef = useRef<{
    renderKey: string;
    todos: TodoItem[] | null;
  } | null>(null);

  if (stableTodosRef.current?.renderKey !== renderKey) {
    stableTodosRef.current = { renderKey, todos };
  }

  return stableTodosRef.current.todos;
}
```

— and the additional `setControlsHeight((prev) => prev === measured.height ? prev : measured.height)` no-op guard at `:1605-1611` and `:1614-1618`, the layout effect only runs and only commits when something layout-relevant actually changed. The new test `does not remeasure footer height for sticky todo status-only updates` at `AppContainer.test.tsx:1426-1462` is the right shape to lock this in: rerender with a `pending → in_progress` flip and assert `mockedMeasureElement.toHaveBeenCalledTimes(callsAfterInitialRender)`.

Two render-key functions were added (`getStickyTodosRenderKey` for shallow-stability of the array contents, `getStickyTodosLayoutKey` for layout-affecting subset including `width` and `maxVisible`). That separation is correct: a status flip touches the render key (and re-renders the panel content) but not the layout key (and skips the footer remeasure).

## Concerns

1. **Recent-history visibility guard is a heuristic.** PR body explicitly acknowledges Ink `<Static>` doesn't expose a per-item viewport API, so the "is the source TodoWrite still likely visible?" check is a conservative bounded-recency approximation. That's fine for the deduplication objective but the threshold (`makeTodoHistory` test fixture has 2 trailing `gemini` items at `:1376-1385`, suggesting the production cutoff is "≤2 items after") is implicit — either pull the magic number out into a named constant in `todoSnapshot.ts` or document it inline. The bug it'll cause if wrong is "sticky panel briefly missing during a fast model response," not data loss, so low risk.
2. **`stickyTodoWidth = Math.min(mainAreaWidth, 64)`** at `:1597` hardcodes the 64-column ceiling. Reasonable for compact rendering, but if a user has a 200-column terminal they'll get a panel that's 64 columns wide while the rest of the chat uses 200 — visually inconsistent. Worth a follow-up issue.
3. **8 files / +463 LOC for a "keep compact" fix** suggests three concerns are bundled (rendering, visibility guard, stability key). Each is correct individually, but a maintainer reading this PR will spend time discriminating "which of these solves the flicker" — splitting into separate commits inside the PR would help.
4. **No macOS/Linux manual repro.** Testing matrix at the PR body shows only Windows manual verification (⚠️ on macOS/Linux). The flicker fix is genuinely cross-platform (Ink's measure-then-commit cycle is the same everywhere), so the risk that it regresses on macOS Terminal/iTerm is low, but a 30-second `npm run` smoke on either would close the loop on the testing matrix.

## Tests

Targeted Vitest covers `todoSnapshot.test.ts`, `StickyTodoList.test.tsx`, two layout test files, plus the new `does not remeasure footer height` regression at `AppContainer.test.tsx:1426`. PR body reports 4 files / 20 tests passing, plus typecheck/prettier/build clean. Coverage looks adequate for the rendering and stability pieces; the visibility guard is also exercised but the heuristic threshold isn't directly asserted.

## What I learned

The flicker root-cause here is the canonical Ink-`<Static>` failure mode: a `useLayoutEffect` whose dep list includes a freshly-allocated array gets re-fired on every render, and PowerShell renders the resulting same-value `setState` as visible flicker because Windows console scrolling is non-buffered. The fix split — *render key* (content changed, redraw the panel) vs *layout key* (geometry changed, remeasure the footer) — is the right discrimination, and the `useRef`-based `useStableStickyTodos` is the smallest possible expression of "promote derived state to a stable identity if its content didn't change". Worth remembering as a pattern for any TUI that mixes streaming state with layout-driven measurement.
