# Review — google-gemini/gemini-cli#26374

- PR: https://github.com/google-gemini/gemini-cli/pull/26374
- Title: perf: optimize performance and memory for large chat sessions
- Head SHA: `0653aa536413451319ae0a62236cb07bcc219805`
- Size: +957 / −233 across 21 files
- Verdict: **merge-after-nits**

## Summary

Performance/memory pass on the TUI for long-running chat sessions:
memoizes the top-level `App` component, extends `historyManager` with
batched insertion (`addItemsBatch`) and active pruning
(`pruneHistory`), wires those into `useMemoryMonitor`, and propagates
the new history surface across the test fixtures.

## Evidence

- `packages/cli/src/ui/App.tsx:15-40` — top-level `App` is wrapped
  in `memo(...)` from `react`, with `App.displayName = 'App'`. This
  is meaningful because `App` re-renders on every UI-state tick from
  `useUIState()`; with `memo` plus the unchanged-props check it now
  only re-renders when its own props (none, currently) change. Small
  detail: since `App` takes no props this is essentially equivalent
  to a stable identity and the perf win comes from the children's
  render boundary, which is the intent.
- `packages/cli/src/ui/AppContainer.tsx:240-244` — `useMemoryMonitor`
  now takes a structured object instead of the whole `historyManager`:
  ```ts
  useMemoryMonitor({
    addItem: historyManager.addItem,
    pruneHistory: historyManager.pruneHistory,
    historyLength: historyManager.history.length,
  });
  ```
  This is the right shape — it narrows what the hook depends on so it
  doesn't re-fire on unrelated `historyManager` mutations.
- `packages/cli/src/ui/App.test.tsx:78-82` and
  `packages/cli/src/ui/components/MainContent.test.tsx:358-390` —
  the `historyManager` mock surface gains `addItemsBatch: vi.fn()`
  and `pruneHistory: vi.fn()`, and `MainContent.test.tsx` adds a
  `renderMainContent` helper that takes a partial `UIState` and
  spreads it onto the default mock. Reasonable pattern.

## Notes / nits

- The `memo` wrapping of `App` only helps if `useUIState()` returns
  a referentially stable value when nothing relevant has changed.
  Worth confirming in the PR body that the underlying state container
  doesn't re-create the object every tick; otherwise `memo` here is
  a no-op (the props comparison would still see the wrapped
  context-provider value re-create).
- `historyManager.history.length` is being passed into
  `useMemoryMonitor`. If the hook reacts to this number, then a
  consumer that mutates history without changing length (rare but
  possible — e.g. in-place edit) won't trigger the monitor. Consider
  also passing a content hash or a bumped revision counter if the
  monitor needs to react to content edits.
- 21 files for a single perf PR is on the large side; would benefit
  from a paragraph in the PR body listing the *measured* before/after
  numbers (memory at N=10k history items, render count per keystroke)
  so reviewers can verify the optimization actually moves the
  needle.

Ship after a quick paragraph of perf numbers in the description.
