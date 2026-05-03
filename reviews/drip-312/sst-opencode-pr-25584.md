# sst/opencode PR #25584 — feat(session): add message-level fork action

- URL: https://github.com/sst/opencode/pull/25584
- Head SHA: `30bc36f6f8cccad34cc6ed24caed3b58cd33d19f`
- Verdict: **merge-after-nits**

## Summary

Wires up a per-message "fork session" UI affordance. The web app gains a
`fork` action plumbed through `actions` to `UserMessageDisplay`, which
exposes a tooltip+icon button. On click it calls
`sdk.client.session.fork({ sessionID, messageID })`, seeds the new
session into `sync.data.session`, copies the source message draft into
the new session's prompt store, and `navigate()`s to the new session
URL.

## Specific references

- `packages/app/src/pages/session.tsx:330` — adds
  `const navigate = useNavigate()`. Standard SolidJS router import,
  matches surrounding style.
- `packages/app/src/pages/session.tsx:1500-1510` — new `seed(next)`
  helper updates `sync.data.session` immutably: replaces the entry if a
  matching `id` already exists (idempotent on retry), otherwise appends.
  Identical pattern to the existing `revert` mutation seed, which is
  good for consistency.
- `packages/app/src/pages/session.tsx:1708-1727` — the `fork(input)`
  mutation. Returns early if `params.dir` is missing (defensive), shows
  a generic `common.requestFailed` toast when `result.data` is missing,
  and on success: seeds, copies the draft via
  `prompt.set(draft(input.messageID), undefined, { dir, id: result.data.id })`,
  then navigates.
- `packages/ui/src/components/message-part.tsx:1064-1077` — the
  `UserMessageDisplay` `fork()` callback. Mirrors the existing `revert`
  shape: bails on `busy()`, sets the busy flag, calls the action, and
  clears in `finally`.
- `packages/ui/src/components/message-part.tsx:1157-1173` — UI: hidden
  unless `props.actions?.fork` is wired, with `aria-label` and tooltip
  using the new `ui.message.forkMessage` i18n key.

## Commentary

This is a clean additive feature. The plumbing follows the established
revert pattern almost line-for-line, which makes review easy and lowers
the regression risk on the unrelated revert path. Hiding the icon
behind `Show when={props.actions?.fork}` means hosts that don't pass
the action in won't see a stub button, which is the right default.

A few things to clean up before merge:

1. **Draft transfer is silent on failure.** `prompt.set(draft(messageID),
   undefined, { dir, id: result.data.id })` assumes the prompt store
   accepts a brand-new session id; if the new session hasn't been
   hydrated yet there could be a race where the draft is written but
   the prompt component reads from the old store. Worth either awaiting
   a hydration signal or at least documenting the assumption near
   `seed`.

2. **No optimistic state for the fork button itself.** `busy()` gates
   re-clicks on the same message, but multiple messages could each
   start their own `fork()` concurrently. That's probably fine at the
   API layer (each call is independent) but it could lead to confusing
   "which fork did the navigate land on" UX if a user mashes two
   buttons. A page-level reverting/forking flag would mirror the
   existing `reverting()` guard used elsewhere in `restore`.

3. **i18n coverage.** Only `ui.message.forkMessage` is added; verify
   the zh-CN/zh-TW/etc. catalogs are updated in the same PR or a
   follow-up. The diff shown only touched the English catalog.

4. **`actions = { revert, fork }`** at line 1741 means every consumer
   of the actions object now sees `fork`. If any host UI relies on
   `Object.keys(actions).length` (unlikely, but possible in feature
   probes), this is a behavior change. Worth grepping.

The core mutation logic is correct and matches existing patterns. None
of the above are blockers — merge after the i18n catalogs land and the
draft-transfer race is either fixed or explicitly acknowledged.
