# sst/opencode PR #25760 — fix: cancel queued messages without aborting session

- Repo: `sst/opencode`
- PR: https://github.com/sst/opencode/pull/25760
- Head SHA: `33d750936565`
- Size: +38 / -3 across 8 files (3 TUI, 3 server, 2 generated SDK)
- Closes upstream issues #4821, #20090.

## Summary

Adds a "Cancel" entry to the per-message dialog when the message is still
queued behind an in-flight assistant turn, so the user can drop the queued
prompt without aborting the active response. The TUI signals queued-ness via
a new `queued?: boolean` prop on `DialogMessage` (`dialog-message.tsx:11-14`)
and the server's `deleteMessage` handler grows an optional `force` query
param so the client can short-circuit the busy-session check it already
knows is safe to skip (`server/routes/instance/session.ts:+8/-1`,
`groups/session.ts:380` adds `query: { force: Schema.optional(QueryBoolean) }`).

## What I like

- The "is this queued?" computation is purely client-side and uses the same
  invariant in two places — `dialog-timeline.tsx:35-40` derives
  `pendingMessageID = messages.findLast((x) => x.role === "assistant" &&
  !x.time.completed)?.id` and compares `message.id > pendingMessageID`;
  `routes/session/index.tsx:1152-1154` does the same against the existing
  `pending()` signal. ULID ordering (lex-sortable, time-prefixed) makes the
  `>` comparison correct, no extra timestamp fetch needed.
- `force` is opt-in and additive on the server. Existing clients that don't
  send it keep the old "fetch all messages, refuse if busy" behavior, so this
  is a backwards-compatible API extension. The handler change is one line
  (`handlers/session.ts:+2/-1`) plumbing the query through.
- Both API surfaces (Hono + HttpApi) are updated, and the SDK regen is
  included (`packages/sdk/js/src/v2/gen/sdk.gen.ts:+2/-0`,
  `types.gen.ts:+1/-0`) so external SDK consumers can call `force: "true"`
  without manual patching.
- The "Cancel" option is conditionally spread (`...(props.queued ? [...] :
  [])` at `dialog-message.tsx:25-41`), so non-queued messages keep the exact
  original action set — no risk of confusing a user who clicks on a
  long-completed message.

## Nits / discussion

1. **`force: "true"` as a string.** The new query param is typed
   `Schema.optional(QueryBoolean)` but the call site at `dialog-message.tsx:34`
   passes `force: "true"` literally. If `QueryBoolean` is the standard
   "accept `true`/`false`/`1`/`0`" coercer, fine — but if it ever tightens
   to require strict booleans, this caller breaks. Worth either a
   typed boolean (`force: true`) on the SDK type or a comment noting the
   coercion contract.

2. **Race between `findLast` and server completion.** Between the moment
   `dialog.replace` opens the message dialog and the moment the user clicks
   "Cancel", the assistant turn may complete — at which point `force=true`
   on a now-non-queued message will delete a real reply mid-stream rather
   than just dropping a queued draft. The server-side check exists exactly
   for this race; bypassing it on the client's word means the worst case is
   "user clicks Cancel on a freshly-finished message and loses the reply."
   Probably acceptable (the dialog is short-lived) but a server-side
   "queued-only" guard — e.g. only honor `force=true` if the message has no
   assistant child yet — would close the window without re-introducing the
   full busy-list scan.

3. **`pendingMessageID` derivation duplicated.** The same expression appears
   in both `dialog-timeline.tsx:35-37` and `routes/session/index.tsx:1152-1154`
   with different inputs (`messages` array vs `pending()` signal). A small
   `isQueued(messageID, pendingMessageID)` helper would prevent these from
   drifting (one already does `findLast`, the other relies on `pending()`
   being current).

4. **Dialog title.** The new option is labeled `"Cancel"` with description
   `"discard queued message"` (`dialog-message.tsx:28-30`). "Cancel" is a
   common dialog verb and risks being misread as "close this dialog";
   `"Discard"` or `"Remove from queue"` matches the description more
   directly. Pure UX nit.

## Verdict

**merge-after-nits.** Clean additive feature, the API extension is
backwards-compatible, and the client-side queued check is correct under
ULID ordering. Worth a follow-up to (a) add a server-side queued-only
guard for the `force=true` path so the client→server race can't delete a
real reply, and (b) consider an `isQueued` helper to dedupe the two
client-side derivations.
