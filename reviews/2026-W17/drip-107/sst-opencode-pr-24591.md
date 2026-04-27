# sst/opencode #24591 — fix: use continuous scroll layout for share page on desktop

- **Repo**: sst/opencode
- **PR**: #24591
- **Author**: workdocyeye
- **Head SHA**: c4b937d97ed715dcd229545e0dd2ede1d2a025f1
- **Base**: dev
- **Size**: +3 / −64 in a single file:
  `packages/enterprise/src/routes/share/[shareID].tsx`.

## What it changes

Closes #18567 by removing the desktop-only "MessageNav sidebar +
single-message focused" layout from the public share page and
collapsing both desktop and mobile to the existing continuous-scroll
`turns()` rendering.

Concretely:

1. The `MessageNav` import at `[shareID].tsx:21` is dropped (the
   component is unused after this PR within the share route).
2. The `<div classList={{ "hidden w-full flex-1 min-h-0": true,
   "md:flex": wide(), "lg:flex": !wide() }}>` desktop branch at the
   former `:282-345` is deleted in its entirety. That branch
   previously rendered:
   - a sticky `MessageNav` for messages with `length > 1`,
   - a `SessionTurn` keyed off `store.messageId ?? firstUserMessage()!.id!`
     (single-message focused view), and
   - a side-by-side `SessionReview` panel when `diffs().length > 0`.
3. The remaining `<Switch>` block that previously rendered a mobile
   `Tabs`-or-scroll fallback (`{ "md:hidden": wide(), "lg:hidden":
   !wide() }`) is now unconditional: the `Tabs` `classList` and the
   else-branch wrapper drop the `md:hidden`/`lg:hidden` keys
   (`:282-318` after the change). `wide()` is no longer consulted
   for layout selection here.

Net effect: every viewport gets `turns()` (continuous scroll) below
the title, with the `Tabs` (`Session` / `Review`) variant when
`diffs().length > 0`. The single-message + sidebar view is gone.

## Strengths

- Pure deletion. No new branches, no new state, no new props. The
  diff is small and the failure modes shrink correspondingly: there
  is no longer a desktop-only render path that can diverge from
  mobile, which was the whole reason MessageNav-vs-continuous parity
  bugs kept showing up.
- The `firstUserMessage()!.id!` non-null assertion at the old `:307`
  is gone. That assertion crashed on shared sessions whose first
  message wasn't a user message (system/assistant-led runs from
  programmatic clients), and avoiding the crash is a side effect
  worth calling out in the PR body.
- Tabs + scroll wrappers no longer carry `wide()`-conditional
  hide/show classes, so `wide()` only governs sidebar/inspector
  width elsewhere. One fewer dimension for a styling regression to
  hide in.

## Risks / nits

- `wide()` is still defined on the route. After this change it's
  consulted only by other render branches (the inspector pane,
  presumably). Worth grepping the file to confirm there's no
  now-orphaned `wide()`-only code path left over; if `wide()` is
  used solely for things outside this file, the variable could be
  trimmed in a follow-up.
- Loss of the deep-link `store.messageId` focus: previously the
  desktop branch keyed `SessionTurn` to `store.messageId`, so a
  shared link like `/share/<id>?messageId=<msg>` jumped straight to
  that message. With continuous-scroll-only, the URL parameter is
  presumably still parsed but no longer drives a different render.
  If `turns()` doesn't scroll-into-view on `store.messageId`, this
  is a behavioral regression for permalinks. Either confirm
  `turns()` already handles `store.messageId` (in which case the
  PR description should say so) or follow up with a small
  scroll-into-view effect keyed off `store.messageId`.
- The `Show when={messages().length > 1}` MessageNav was the only
  thing showing the user a list of conversation turns at a glance
  on desktop. Removing it without a replacement reduces
  navigability for long shared sessions. Not a blocker — the
  continuous scroll matches mobile and the rest of the app — but
  a future "jump to message" affordance would close the regression
  for users who relied on it.
- `MessageNav` is still exported from
  `@opencode-ai/ui/message-nav`. If this share route was the last
  consumer, the package barrel could be slimmed in a follow-up; if
  it's used elsewhere, no action needed.
- The `Show when={diffs().length > 0}` side-by-side review pane is
  also gone from desktop. On a wide monitor users now have to
  switch tabs (`Session` / `Review`) instead of seeing both
  simultaneously. Worth confirming the issue (#18567) actually asks
  for that or whether it was a side-effect of removing the focused
  layout.

## Suggestions

- In the PR body, explicitly enumerate the *behavioral* regressions
  (no MessageNav on desktop, no side-by-side review pane on
  desktop, deep-link `messageId` no longer drives focus) so a
  reviewer who only looks at the diff understands the user-visible
  cost.
- If `store.messageId` is still parsed but no longer used for
  rendering, add a small `createEffect` that scrolls the matching
  turn into view on mount, so permalinks stay useful.
- Consider a follow-up that re-introduces a *collapsed*
  table-of-contents affordance (overlay or popover triggered from
  the title bar) so long shared sessions remain navigable without
  the old sidebar.

## Verdict

`merge-after-nits` — the simplification is correct and removes a
known divergence between desktop and mobile renderers, but the
`store.messageId` permalink behavior and the silent loss of the
side-by-side review pane on desktop should be acknowledged
explicitly (in the PR body, in commit, or with a small
follow-up issue) before merging.
