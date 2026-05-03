# Review: sst/opencode #25634 — Improve v2 session message rendering

- PR: https://github.com/sst/opencode/pull/25634
- Head SHA: `fed24cbda8b8b9e3b8c4638ef5062417813aae5c`
- Author: thdxr
- Files touched (2):
  - `packages/opencode/src/cli/cmd/tui/context/sync-v2.tsx` (+8 / -8)
  - `packages/opencode/src/cli/cmd/tui/feature-plugins/system/session-v2.tsx` (+108 / -69)

## Verdict: `request-changes`

## Rationale

The PR flips message buffer ordering in `sync-v2.tsx` from `findLastIndex`/`push` to `findIndex`/`unshift` so that `messages[0]` becomes the most recent entry, then drops the now-redundant `.toReversed()` semantics in the view. That's defensible, but the changeset is internally inconsistent: `session-v2.tsx:46` still calls `renderedMessages = createMemo(() => messages().toReversed())` even though the underlying array is now already in reverse-chronological order. After this PR, `renderedMessages()` is back to chronological, which contradicts the new `lastUserCreated` helper added at `session-v2.tsx:47-50` — that helper does `.slice(0, index).findLast(... "user")` which only makes sense if the array is reverse-chronological (looking *before* index for the next-older user message). With the leftover `.toReversed()`, the helper picks the newest user message *after* the current assistant, not the user message that triggered it, which is the opposite of the stated goal of pairing `start` time. Either drop the `.toReversed()` (matching the new sync ordering) or keep ordering as-is and revert the sync-v2 changes; can't have both. Additionally, the diff completely removes the `SyntheticMessage` component (it's now `<></>` at line ~93) but leaves the `session.next.synthetic` event still calling `unshift({ type: "synthetic", ... })` in sync-v2.tsx — synthetic messages will now silently accumulate in state with no UI representation. Either delete the event handler too or render something. Worth a second pass before merging.