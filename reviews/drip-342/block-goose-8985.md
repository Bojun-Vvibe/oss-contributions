# block/goose #8985 — refactor: goose 2 ui used acp session id

- **Head SHA:** `c58787912640343e1ab4a954521607bad1b58a2f`
- **Size:** +304 / -750 across many files
- **Verdict:** **merge-after-nits**

## Summary
Net deletion (-446 LOC) that removes the parallel `gooseSessionId`
bookkeeping in the goose2 UI: callers used to look up an ACP-side session ID
via `getGooseSessionId(sessionId, personaId)` and pass it as a second
argument to `acpLoadSession` and friends. After this change the UI simply
uses the local `sessionId` everywhere — `acpLoadSession(sessionId,
workingDir)` (`ui/goose2/src/app/AppShell.tsx:117`),
`acpCancelSession(sessionId)` (`useChat.ts:313`), and `getGooseSessionId` /
`streamingPersonaIdRef` are deleted. Tests are updated to drop the
`personaId`/`gooseSessionId` arguments throughout
(`useChat.compaction.test.ts`, `useChat.personaPreparation.test.ts`,
`useChatSessionController.test.ts:194,348`).

## Strengths
- This is the rare refactor that *removes* state. The dual identity
  (`sessionId` ↔ `gooseSessionId` per-persona) was real complexity and a
  real source of bugs whenever the two drifted; collapsing to one ID is the
  right direction.
- Test updates are mechanical and exhaustive — every assertion that
  previously checked for the second argument now checks for its absence.
  The compaction test specifically asserts
  `mockAcpLoadSession` is called with two args, not three
  (`useChat.compaction.test.ts:88`), which would have caught any caller this
  PR forgot to migrate.
- `streamingPersonaIdRef` deletion (`useChat.ts:108-109, 226, 290, 304`) is
  done in one pass and the dependent `getStreamingPersonaId` callback
  (lines 116-126) is deleted with it. No dangling refs.
- Compaction path simplification (`useChat.ts:386` onward in the diff): the
  PR removes the "look up gooseSessionId, fall back to ensurePrepared, look
  up again" dance and replaces it with a single `setChatState(sessionId,
  "compacting")` plus the underlying ACP call. This is the kind of
  conditional ladder that quietly grows over time, and removing it is a
  good thing.

## Concerns / nits
- `acpCancelSession(sessionId)` no longer carries the persona ID
  (`useChat.ts:313`). The previous test
  `keeps persona-aware cancellation working after remount` is *deleted* in
  this PR (visible in the diff around the goose deletion block). That test
  existed for a reason — if the backend was previously routing cancellation
  by persona, dropping that parameter quietly may cancel the wrong stream
  when two personas share a session. Worth confirming in the PR description
  that the backend now derives the persona from the session ID and the
  removed test is intentionally obsolete (not just inconvenient).
- Same for the deletion in `AppShell.tsx:199, 467` — the `{ personaId: ...
  }` option is removed from session-create calls. If the ACP side stored
  persona binding at create time, this is a behavior change, not a
  refactor. The PR description does not say which.
- The new compaction error path
  (`useChat.compaction.test.ts:235-247`, "surfaces an error when preparing
  for compaction fails") replaces the previous "surfaces an error when
  compacting before the session is prepared" test. Net the user-visible
  contract is similar, but the error surface changed from
  "session-not-prepared" to "ensurePrepared rejected". A migration note
  would help.
- The diff is large enough that a reviewer needs to spot-check that no
  call site of `acpLoadSession` / `acpCancelSession` was missed. A quick
  `git grep getGooseSessionId` left no hits → likely complete, but worth
  confirming in the PR description.

## Recommendation
Land after the PR description explicitly answers: (1) why dropping
`personaId` from cancel/create is safe, (2) what happens when two personas
share one session, and (3) whether any backend change depends on this UI
change shipping in the same release. The simplification is good; the
implicit contract change is the part that needs sign-off.
