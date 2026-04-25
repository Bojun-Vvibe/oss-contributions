# Review: All-Hands-AI/OpenHands#14105 — Allow send when only attachments are present

- **PR**: https://github.com/All-Hands-AI/OpenHands/pull/14105
- **State**: OPEN
- **Author**: he-yufeng (Yufeng He)
- **Range**: +5 / −1
- **Head SHA**: `c1c2828372c77b9fcb67dfb15cbe204a0f009780`
- **Base SHA**: `1091901be206279bdb755065f72c6f4f415fb3ec`
- **Fixes**: #14047
- **Verdict**: merge-after-nits
- **Reviewer date**: 2026-04-25

## What the PR does

`frontend/src/hooks/chat/use-chat-submission.ts` previously
short-circuited `handleSubmit` whenever the trimmed text was
empty, silently swallowing image-only or file-only
submissions. The fix pulls `images` and `files` from
`useConversationStore` and only bails out when **all three**
(text, images, files) are empty:

```ts
const { images, files } = useConversationStore.getState();
if (!trimmedMessage && images.length === 0 && files.length === 0) {
  return;
}
```

## Observations

1. **The diagnosis matches the diff.** The interactive chat
   box already forwards attachments to the `onSubmit` arrow at
   the call site and clears them via `clearAllFiles()` after
   submit (per PR body). The break was specifically the
   trimmed-text guard at the top of `handleSubmit`. Lifting
   the guard to consider attachments closes the loop without
   changing the downstream submit shape.
2. **`useConversationStore.getState()` (not the hook) is the
   correct call here.** Using `useConversationStore(...)` would
   re-render the component on every store change; calling
   `.getState()` inside a callback is the Zustand idiom for
   "read once at submit time". Right call.
3. **Race window is small but exists.** Between the user
   clicking send and `getState()` resolving, an attachment
   removal (e.g. user clicks the "x" on an image just as
   they hit Enter) could read stale state and submit the
   removed attachment. In practice the `clearAllFiles()`
   after submit handles this, but worth a follow-up issue if
   anyone reports phantom attachments.
4. **No regression on the truly-empty case.** The `&&`
   conjunction preserves the original guard semantics for
   no-text + no-attachments, which the PR body explicitly
   verifies in the "to verify" list ("Empty text + no
   attachments → send button still does nothing"). Good
   defensive note.
5. **Test coverage gap.** This is a frontend hook with
   straightforward state access — it should have a vitest /
   jest test asserting the four cases (text only, image
   only, file only, all empty). The fix is mechanically
   correct, but the absence of a test means a future
   refactor of `useConversationStore`'s shape silently
   re-breaks this. Would block merge on adding a 4-case
   table-driven test for `useChatSubmission`.
6. **PR author honestly flags they couldn't run V1 dev
   stack.** Manual smoke list ("attach only image →
   send / press Enter / etc.") is the right escalation —
   relies on a reviewer with the dev env. Maintainers
   should make sure that smoke pass actually happens before
   merging.

## Verdict reasoning

Right fix for the right bug, narrow blast radius, store
access uses the right idiom. Two nits: add the 4-case unit
test, and confirm the smoke list actually got run by a
reviewer with V1 dev env. With those, ship it.

## What I learned

Empty-text guards on chat submission are a category of bug
that recurs whenever an "attachment" surface is added to an
existing text-only flow. The defensive pattern is to define
"is empty" at the *intent* layer (no text and no
attachments and no recordings and ...) rather than at the
text layer, so adding a new attachment type doesn't silently
re-introduce the bug. A typed `hasContent(input)` helper
would future-proof this against the next attachment kind.
