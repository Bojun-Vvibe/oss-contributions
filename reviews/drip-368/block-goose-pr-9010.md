# block/goose PR #9010 — preserve workspace context for project chats

- Head SHA: `3e1c7bc96ea49d0ead6b96aa0d72a261a4e445f1`
- Author: `morgmart`
- Size: +144 / -10
- Verdict: **merge-after-nits**

## Summary

Fixes a real "the new chat lost my workspace" UX bug in the
goose2 desktop UI: when a user has chat A open in project P
with active workspace `/repo/feature` (a non-default working
directory inside the project's set of allowed roots), then
clicks "new chat" *within the same project*, chat B was
spawning back at the project's first/default workspace
(`projectWorkingDirs[0]`) instead of inheriting `/repo/feature`.
The fix introduces a small helper
`resolveInheritedProjectWorkspace` that captures the active
chat's workspace selection (with a session.workingDir fallback
for post-reload state) and threads it through both the existing-
draft and new-session creation paths.

## What the diff actually does

1. **New helper** at
   `ui/goose2/src/features/chat/lib/workspaceContext.ts:1-38` —
   pure function that returns the active chat's
   `ActiveWorkspace` if (a) there's a `projectId` and an
   `activeSessionId`, (b) the active session is in the same
   project, and (c) either the active workspace store has an
   entry for it, or (fallback) the active session has a
   `workingDir` field. The cross-project check at line 27
   (`activeSession?.projectId !== projectId`) is the right
   guard — it prevents inheriting a workspace from project P
   when creating a chat in project Q.
2. **AppShell.tsx:286-302,316-336** — the new-chat handler
   computes `inheritedWorkspace` once, then in **both** the
   "reuse existing draft" branch (lines 304-313) **and** the
   "create new session" branch (lines 319-336) calls
   `setActiveWorkspace` and `updateSession({ workingDir })` so
   the inherited path is persisted both to the in-memory active
   workspace map *and* to the session's `workingDir` field
   (the second is what survives reload).
3. **AppShell.tsx:115-122** — session-load path now reads
   `activeWorkspaceBySession[session.id]` and passes its path
   (or session.workingDir as fallback) into `resolveSessionCwd`,
   so opening an existing chat respects its last-set workspace
   instead of always falling back to the project default.
4. **AppShell.tsx:494** — after the ACP project-switch refresh,
   the resolved `workingDir` is now persisted back via
   `sessionStore.updateSession(sessionId, { workingDir })`,
   keeping the on-disk session record in sync with the runtime
   state. This is the small but important "now we don't lose
   the choice across reload" piece.
5. **useChat.ts:367-372** — `getWorkingDir` now falls back to
   `sessionStore.getSession(sessionId)?.workingDir` when the
   active workspace map has no entry, which is the post-reload
   state where the in-memory map starts empty but the persisted
   session record still has the last working dir.
6. **useChatSessionController.ts:170** — analogous persistence
   on the project-switch path (`updateSession(sessionId, {
   workingDir })` after `acpPrepareSession`), so a
   manually-triggered project switch also persists the
   resolved cwd.
7. **ContextPanel + WorkspaceWidget + ChatContextPanel +
   ChatView** — adds `sessionWorkingDir` prop threading from
   `controller.session?.workingDir` (set in `ChatView.tsx:185`)
   down through `ChatContextPanel.tsx:34,70` and
   `ContextPanel.tsx:39,52,186`, with the `gitTargetPath`
   precedence chain at `ContextPanel.tsx:51-52` becoming
   `activeContext?.path ?? sessionWorkingDir ??
   projectDefaultWorkspaceRoot`. The same precedence applies in
   `WorkspaceWidget.tsx:63-64` for the displayed primary
   workspace root. This is the read-side companion to the
   write-side persistence above — git context now reflects the
   inherited workspace immediately rather than waiting for the
   user to re-pick.
8. **Test** at
   `ui/goose2/src/features/chat/lib/workspaceContext.test.ts:1-55` —
   three vitest cases covering the three branches: (a) inherits
   from active workspace map, (b) falls back to
   `activeSession.workingDir` when the map is empty
   (post-reload), (c) returns `undefined` across projects. The
   third case is the critical one — it pins the cross-project
   isolation invariant. Test fixture is well-shaped via a
   `makeSession` helper.

## Why merge-after-nits

The design is correct, the precedence chains line up, and the
helper is well-tested in isolation. Concerns are integration-
shaped, not correctness-shaped:

1. **No integration test for the AppShell handler.** The
   helper has unit tests but the actual reuse-existing-draft
   and create-new-session paths in `AppShell.tsx:286-336` —
   where the helper's output is dispatched to two separate
   store mutations with non-trivial ordering — has no
   end-to-end test. A vitest case that mounts AppShell with a
   project + active chat + active workspace, clicks "new chat",
   and asserts the new chat's `workingDir` matches the active
   workspace path would lock down the integration. The
   workspaceContext.test.ts tests pass even if the AppShell
   never calls the helper.

2. **Race between `setActiveWorkspace` and `updateSession`** at
   `AppShell.tsx:304-308`. Both are sync zustand store calls so
   no actual race, but the order matters for any selector that
   joins both stores: if a downstream selector reads the
   session.workingDir first then derives gitTargetPath via
   active-workspace-or-fallback, you get a transient state
   where workingDir = inherited but activeWorkspace = stale.
   In practice React batches and selectors recompute together,
   but it's worth a comment at the call site or, cleaner,
   wrapping both writes in a single zustand `setState` call so
   subscribers see the joint mutation.

3. **`session.workingDir` semantics drift.** Pre-PR,
   `session.workingDir` was set once at session creation and
   was effectively immutable (the active-workspace map carried
   the live state). Post-PR, `session.workingDir` is now
   actively mutated on (a) project switch
   (`AppShell.tsx:494`), (b) new-chat-with-existing-draft
   (`AppShell.tsx:306`), and (c) `useChatSessionController`
   workspace selection (`:170`). This is the right direction
   but it's a meaningful semantic shift that should be called
   out in a comment near the `ChatSession.workingDir` type
   definition so future maintainers don't assume immutability.

4. **`resolveInheritedProjectWorkspace` returns `branch:
   null` in the fallback path** (line 35 of `workspaceContext.ts`).
   That's plausibly correct (we don't know the branch from the
   raw `workingDir` string) but `setActiveWorkspace` now
   receives `{ path, branch: null }` which may differ from
   what a fresh git-state read would produce
   (`branch: 'feature'`). If `ActiveWorkspace.branch` is used
   anywhere as a "trust this, don't re-fetch" cache key, the
   `null` could cause a stale display. Verify or, safer, omit
   `branch` from the fallback object so consumers always treat
   it as "branch unknown, refetch" rather than "branch is
   null".

5. **The `setActiveWorkspace` + `updateSession` pair appears
   in three places** (AppShell.tsx:306, AppShell.tsx:330-332
   via separate calls, and useChatSessionController.ts:170).
   Worth extracting a `persistInheritedWorkspace(sessionId,
   workspace)` helper to ensure the two writes don't drift —
   if a future change adds e.g. a `lastWorkspaceUpdatedAt`
   timestamp, all three sites should update it.

## Optional nits (not blocking)

- `useChat.ts:367-372` — the `??` chain returns
  `undefined` when both the active workspace and session
  workingDir are missing. Consumers downstream may already
  handle `undefined` correctly, but a fallback to the project
  default would match the read-side precedence chain in
  `ContextPanel`.
- The PR title `preserve workspace context for project chats`
  is accurate but slightly underspecified — `fix(chat):
  inherit active workspace when creating new chat in same
  project` would scan better in `git log`.
- The new helper file lives at
  `features/chat/lib/workspaceContext.ts` while the related
  store is at `features/chat/stores/chatSessionStore.ts` —
  consistent with existing layout, no action needed, just
  noting the file is well-placed.
