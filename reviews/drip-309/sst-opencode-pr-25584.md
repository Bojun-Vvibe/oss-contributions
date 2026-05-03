# sst/opencode #25584 — feat(session): add message-level fork action

- PR: https://github.com/sst/opencode/pull/25584
- Author: adrian15
- Head SHA: `8f5ef02e44c5c142e36a4feced6b95fb2490ee16`
- Updated: 2026-05-03T12:31:48Z

## Summary
Adds a per-user-message "fork" action to the session UI. The handler calls `sdk.client.session.fork({ sessionID, messageID })`, seeds the new session into the local sync store, copies the message's draft into the new session's prompt slot, and navigates to the forked session.

## Observations
- `packages/app/src/pages/session.tsx:1497-1506` (`seed`): inserts-or-replaces the new session in `sync.data.session`. Returning `[...list, next]` when the session is not yet present means a fresh fork lands at the bottom of the session list rather than at the top, which is inconsistent with how new sessions are normally surfaced. Most users expect the just-forked session to be visible at the top — consider `[next, ...list]` or wherever the list's canonical sort lives.
- `packages/app/src/pages/session.tsx:1708-1725` (`fork`): silently `return`s when `params.dir` is missing without surfacing why. The button is shown when `props.actions?.fork` is truthy, so a user could click it from a state where `dir` is undefined and nothing visible happens. Either toast a `requestFailed`, or guard the button by also requiring `dir`.
- `packages/app/src/pages/session.tsx:1715-1719`: `if (!result.data) { showToast(... requestFailed) ; return }` — good. But the SDK call itself can reject (network, 4xx); the chain ends in `.catch(fail)`. Verify `fail` (not shown in diff) emits a user-visible toast — silent network failures on this action would be confusing.
- `packages/app/src/pages/session.tsx:1717`: `prompt.set(draft(input.messageID), undefined, { dir, id: result.data.id })`. The intent is "carry the user's draft text from the original message into the new session's prompt." Confirm `draft(messageID)` is keyed by the *original* message id and that the new session's prompt slot keyed by `result.data.id` is empty before this runs (otherwise the user could lose an existing draft in the destination session — unlikely since it is brand new, but worth a unit test).
- `packages/ui/src/components/message-part.tsx:1064-1075` (`fork` callback): wraps `act(...)` in `Promise.resolve().then(...)`, then `.finally(() => setState("busy", false))`. Mirrors the existing revert pattern — good. However `revert` (above this block, not in diff) likely awaits the same way; if revert ever throws synchronously the `.finally` covers it. Consistency is fine.
- `packages/ui/src/components/message-part.tsx:1157-1172`: new fork `IconButton` is gated on `props.actions?.fork`. The `Tooltip`/`aria-label` use `i18n.t("ui.message.forkMessage")`. The diff does not show the i18n string being added to the locale catalogs — if locale JSON files aren't updated in this PR, the tooltip will render the raw key. Add the `forkMessage` key to at least the default locale.
- No backend test for `session.fork` is in the diff; assuming the route already exists (the SDK type is referenced via `Awaited<ReturnType<typeof sdk.client.session.fork>>`). If `session.fork` is also new, this PR is undersized for what it claims; please confirm.
- No icon registration for `icon="fork"` shown — verify the icon is registered in the shared icon set so the button doesn't render blank.

## Verdict
`merge-after-nits`
