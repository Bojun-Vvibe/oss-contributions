# Review: block/goose#9049 — chore: goose2 UI state refactor (part 1)

- Head SHA: `e3656c9a706f4a47cef0e850880d2cc2606ab70e`
- Files: 38 (+2200 / -337)
- Verdict: **needs-discussion**

## Summary

Part 1 of a multi-PR refactor migrating the goose2 UI from broad Zustand store subscriptions to targeted selectors. Adds selector helper modules for chat / chatSession / agent / project stores, replaces the generic `chatSessionStore.updateSession` with `patchSession` (local-only) plus explicit backend-aware `updateSessionTitle` / `updateSessionProject` operations, and removes two pieces of dead code (`findLastIndex`, unused skill retry payload helper).

## Specific evidence

- **Broad-subscription → targeted-selector swap** at `ui/goose2/src/app/AppShell.tsx:131-148`:
  ```ts
  // BEFORE
  const chatStore = useChatStore();
  const sessionStore = useChatSessionStore();
  const agentStore = useAgentStore();
  const projectStore = useProjectStore();
  // AFTER
  const messagesBySession = useChatStore(selectMessagesBySession);
  const sessions = useChatSessionStore(selectSessions);
  const activeSessionId = useChatSessionStore(selectActiveSessionId);
  const hasHydratedSessions = useChatSessionStore(selectHasHydratedSessions);
  const sessionsLoading = useChatSessionStore(selectSessionsLoading);
  // ... 12 more targeted hooks ...
  ```
  This is the right Zustand pattern — broad `useStore()` re-renders on every state slice change; targeted selectors only re-render on the selected slice. Net win, but `AppShell` now has 14+ targeted hooks at the top of one component, which strains readability.
- **New selector modules**:
  - `features/agents/stores/agentSelectors.ts` (+9)
  - `features/chat/stores/chatSelectors.ts` (+7)
  - `features/chat/stores/chatSessionSelectors.ts` (+12)
  - `features/projects/stores/projectSelectors.ts` (+3)
  Tiny modules — selector-per-field pattern. Good for tree-shaking, but at 9 lines for `agentSelectors.ts` the file overhead is high relative to content. Worth considering one selectors module per store rather than per-field exports.
- **Backend-aware session ops** at `features/chat/stores/chatSessionOperations.ts` (+28, new file): `updateSessionTitle`, `updateSessionProject`. The naming makes the backend-mutation intent explicit vs the new local-only `patchSession`. Tested at `__tests__/chatSessionOperations.test.ts` (+114, new file).
- **Local-only `patchSession`** at `features/chat/stores/chatSessionStore.ts:2/-27` — net subtractive, replaces the generic `updateSession` that was both mutating local state and (sometimes) firing a backend call depending on the caller. The split into `patchSession` (local) + `updateSession*` (backend-aware) makes the side-effect model explicit per call site.
- **Dead-code removal**:
  - `features/chat/lib/skillSendPayload.ts` deleted (`+0/-17`).
  - `shared/lib/arrays.ts` `findLastIndex` deleted (`+0/-10`) — `Array.prototype.findLastIndex` is now standard, so the polyfill/helper is redundant.
- **1700+ of the 2200 added lines are docs**: `ui_improvements/state_management/*.md` — 11 markdown files (`goose2-zustand-state-management-improvement-plan.md` 464 lines, `goose2-zustand-state-management-review.md` 384 lines, plus 9 phase docs at 51-101 lines each, plus `refactor-progress.md` 154). PR body acknowledges these will be removed when refactoring is complete.
- **Test impact**: `Sidebar.test.tsx` updated (+16/-15) — symmetric churn matching new selector signatures. `chatSessionStore.test.ts` shrinks from -32 to +10 (net -22), suggesting some store-level invariants moved into the new operations layer's tests.

## Why needs-discussion (not request-changes)

Three concerns warrant maintainer alignment before this lands:

1. **1700-line in-tree refactoring plan as committed code.** Plans/checklists belong in an issue, a wiki, or `.github/ROADMAP.md`, not as `.md` files inside `ui/goose2/ui_improvements/state_management/`. The PR body promises removal "when the refactoring is completed" — that's at least 8 phases away (phases 1-9 in the plan), so these files will live in `main` for an indeterminate window. Question for maintainers: is in-tree planning the team's preferred convention, or should these move to a tracking issue before merge?
2. **`AppShell.tsx` selector explosion.** 14+ `useStore(selectX)` calls at the top of one component is a code smell that the next phase should address (likely a custom hook bundling the AppShell-relevant slice). Worth confirming the next-phase plan addresses this rather than leaving it as a long-lived state.
3. **Backward-compat surface for `updateSession`.** Removing the generic `updateSession` from `chatSessionStore` is a breaking change for any external consumer (skill, plugin, hook) that imported it. The PR is internal-only refactor per the file list, but a `@deprecated`-and-shim would harden the migration; outright removal needs a maintainer signoff that no plugin surface depends on it.

The actual code change is sound — Zustand selector pattern, explicit side-effect boundary on session mutations, dead-code removal. The verdict is needs-discussion solely because the in-tree plan-doc convention and the breaking removal need maintainer alignment before merge, not because the code is wrong.
