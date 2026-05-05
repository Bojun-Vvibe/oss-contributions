# block/goose #9014 — show unread state for background chat responses

- **Head SHA:** `ec224a170d8196c831481b33aee588e2533a0efe`
- **Base:** `main`
- **Author:** morgmart
- **Size:** +184 / −27 across 7 files (1 AppShell, 1 chatStore, 1 chatStore tests, 1 acpNotificationHandler, 2 acpNotificationHandler tests, 1 SessionActivityIndicator tests)
- **Verdict:** `merge-after-nits`

## Summary

Closes the gap where the goose2 sidebar would show an active chat as "busy"
while the agent was working but had no signal once the response finished if
the user was looking at a different view. Introduces a `viewedSessionId`
state slot in `chatStore` (distinct from the existing `activeSessionId` —
"selected" vs. "actually visible right now") and routes every
`activeView` change through a `setActiveViewWithViewedSession` callback that
keeps the two in sync. The ACP notification handler then marks live agent
output unread when it arrives for any session that isn't currently being
viewed, while leaving replayed history (during session-load) silent.

## What's right

- **`viewedSessionId` is a real new state, not a re-purposing of
  `activeSessionId`.** The PR keeps the existing semantic of
  `activeSessionId` ("which session is selected") and adds
  `viewedSessionId` ("which session is on screen right now") at
  `chatStore.ts:46`. The two only diverge when the user navigates to a
  non-chat view (settings, home) while a chat session remains "selected" —
  exactly the case the bug occurred in.

- **`useLayoutEffect` not `useEffect`.** At `AppShell.tsx:172-174` the
  `syncViewedSessionForView` call is in a `useLayoutEffect`, which fires
  before the browser paints. This avoids a one-frame window where the
  unread indicator could flash on a session the user is about to view.
  Subtle and correct.

- **Replay-vs-live distinction is the right boundary.** The unread-mark
  logic in `acpNotificationHandler.ts` (the +10 lines around `loadingSessionIds`)
  fires only when the session is NOT in `loadingSessionIds`, so the
  replay path during initial session load does not falsely accumulate
  unread state. Pinned by the test at
  `acpNotificationHandler.test.ts:209-226` ("does not mark replayed agent
  output unread") which sets `loadingSessionIds: new Set(["acp-session"])`
  and asserts `hasUnread === false`.

- **Cleanup symmetry.** `cleanupSession` at `chatStore.ts:513-516` is
  updated to also clear `viewedSessionId` if it matches the cleaned-up
  session, mirroring the existing `activeSessionId` clear. Pinned by the
  test at `chatStore.test.ts:189-194`. Without this, a session-deletion
  flow would leave a stale `viewedSessionId` pointing at a deleted
  session.

- **Three-way unread coverage.** The new tests at
  `acpNotificationHandler.test.ts:147-228` pin all three relevant cases:
  (a) unviewed session → unread, (b) viewed session → not unread, (c)
  active session that is NOT being viewed (because activeView is
  settings/home) → unread. Case (c) is the bug-fix test.

- **`setActiveViewWithViewedSession` is the only public mutator.** Every
  call site that previously did `setActiveView("chat")` now goes through
  the wrapper (8 sites in `AppShell.tsx`: `:316`, `:334`, `:402`, `:450`,
  `:533`, `:541`, `:573`, `:657`, `:687`). This eliminates the class of
  bug where a future contributor adds a 9th `setActiveView` call site
  and forgets to sync `viewedSessionId`. The `useCallback` dep arrays
  are kept in lockstep — every site that depends on
  `setActiveViewWithViewedSession` lists it explicitly.

## Concerns / nits

1. **`useChatSessionStore.getState()` direct read inside `handleNavigate`
   at `AppShell.tsx:567-570`.** The `nextSessionId` is computed via
   `view === "chat" ? useChatSessionStore.getState().activeSessionId :
   null`. This is a non-reactive read — if the active session changes
   between handler creation and handler invocation, the right value is
   read at invocation time, which is what we want. But the pattern of
   "imperative store getState() inside an event handler" is unusual in
   this codebase; a one-line comment explaining why this is intentional
   (and not a forgotten `useChatSessionStore(state => state.activeSessionId)`
   subscription) would help future readers.

2. **`syncViewedSessionForView` is a free function that calls
   `useChatStore.getState()` twice (lines 73-79).** The first
   `getState()` gets the store, then `liveChatStore.setViewedSession`
   and `liveChatStore.markSessionRead` are called on the snapshot. If
   `setViewedSession` ever becomes a setter that swaps the store
   (rather than merging), the `markSessionRead` call would fire on a
   stale snapshot. Minor but worth either capturing the actions
   explicitly (`const { setViewedSession, markSessionRead } =
   useChatStore.getState()`) or re-fetching `useChatStore.getState()`
   between the two calls.

3. **`SessionActivityIndicator.test.tsx:+14`** lines added are not in the
   visible diff but the PR description claims "Adds coverage that the
   running spinner takes priority while active and the unread indicator
   appears after activity ends." The "spinner takes priority" assertion
   is the visual rule the user actually cares about — please verify the
   test pins the priority order (spinner wins over unread dot when both
   conditions hold) and not just the standalone presence of each
   indicator.

4. **`viewedSessionId: null` reset is added to four test setup blocks**
   (`acpNotificationHandler.test.ts:43`, `chatStore.test.ts:30 / :207 /
   :223`). All four are correct — without these, a test that calls
   `setViewedSession` would leak state into the next test. But this
   means every test file that uses `useChatStore` and adds new state
   to it now needs to remember to reset both `activeSessionId` AND
   `viewedSessionId` (and any future state). Consider extracting a
   `resetChatStore()` helper in `chatStore.ts`'s test-utils so future
   state additions don't require updating N test files.

5. **`useLayoutEffect` deps are `[activeSessionId, activeView]`** at
   `AppShell.tsx:174`. Correct — but `syncViewedSessionForView` itself
   is a free function defined outside the component (lines 70-79) so
   it's stable by reference and doesn't need to be in deps. The lint
   rule `react-hooks/exhaustive-deps` should be happy with this; just
   confirming the choice is intentional and not a missed dep.

6. **The `cleanupSession` test at `chatStore.test.ts:172-194` calls
   `store.cleanupSession("s1")` after destructuring `store = ...`, but
   then asserts on `useChatStore.getState()`** at line 184 ("`const
   state = useChatStore.getState()`"). The mid-test refresh is correct
   (the destructured `store` snapshot is stale after `cleanupSession`)
   but the previous in-place `expect(store.X).toBe(...)` assertions at
   `chatStore.test.ts:185-188` use `state.X` consistently — good. Just
   note that the previous unrelated assertions at lines 185-188 changed
   to use `state.` which is correct fix; verifying they passed before
   relied on Zustand's snapshot semantics.

7. **`markSessionUnread` is called from
   `acpNotificationHandler.ts:+10`** (not in visible diff). Verify this
   call only fires for `agent_message_chunk` and `tool_use_*` updates,
   not for every notification kind — a `session_metadata_update` (or
   similar) shouldn't bump unread. The four new tests cover
   `agent_message_chunk` only; a fifth test asserting that, e.g., a
   thought-only update for a non-viewed session does NOT mark unread
   would close the loop.

## Verdict rationale

Real UX bug (background chats finishing silently with no badge), narrow
fix scoped to the exact gap, tests cover the three relevant cases plus
the cleanup symmetry. Concerns are stylistic / test-coverage polish —
none block merge.
