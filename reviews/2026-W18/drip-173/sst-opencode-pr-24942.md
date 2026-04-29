# sst/opencode #24942 — show tool parameters during pending state

- **PR:** https://github.com/sst/opencode/pull/24942
- **Title:** `fix(ui): show tool parameters during pending state`
- **Author:** mrsimpson
- **Head SHA:** 5b44d10adf58ec937f34585e7929613bd5dc937b
- **Files changed:** 1 (`packages/ui/src/components/basic-tool.tsx`), +32 / −34
- **Verdict:** `merge-as-is`

## What it does

Removes the `!pending()` gate that was hiding tool subtitle/args/action and
the collapsible details arrow while a tool call is awaiting user
permission. Before this change, the user was asked to approve a tool call
without seeing the very arguments they were approving. Fixes #24934.

## Why this is correct

Pre-change (`basic-tool.tsx:148`):

```tsx
<Show when={!pending()}>
  <Show when={title().subtitle}>...</Show>
  <Show when={title().args?.length}>...</Show>
</Show>
...
<Show when={!pending() && title().action}>...
<Show when={props.children && !props.hideDetails && !props.locked && !pending()}>
  <Collapsible.Arrow />
</Show>
```

Post-change: the outer `<Show when={!pending()}>` and the `!pending()`
clauses on `action` and `Collapsible.Arrow` are dropped. Net: subtitle,
args, action, and the expand arrow are now visible during the
`pending`/`running` shimmer state. The outer `pending()`-driven
`TextShimmer` on the title remains (`basic-tool.tsx:144`), so the visual
"in progress" affordance is preserved — only the *information hiding* is
removed.

## What's good

- Minimum-blast-radius fix. One file, ~33 lines net, no new state, no
  behavior change for non-pending rows.
- Correctly leaves `props.locked` and `props.hideDetails` as the
  remaining gates on the collapsible arrow — those are the semantic gates
  for "user shouldn't expand this", whereas `!pending()` was incidental.
- Aligns with the security model: a permission prompt MUST display the
  exact payload being approved. Hiding args at approval time is a
  well-known footgun (you approve `Bash` and only later learn it was
  `rm -rf /`).

## Nits / risks

- No screenshot or before/after in the PR body. A reviewer who hasn't
  seen the regression has to trust the description that
  shimmer + visible args reads as "pending" rather than "ready". From
  the diff the shimmer animation is preserved on the title text itself
  (line ~144), so this should be fine, but worth a maintainer eyeball
  in dark mode where shimmer contrast is weaker.
- The `onClick` on the subtitle is still wired during `pending` —
  `e.stopPropagation()` and `props.onSubtitleClick()` will now fire
  while a tool is awaiting approval (`basic-tool.tsx:154-159`). For most
  current callers `onSubtitleClick` is a "navigate to the file" handler,
  which is harmless; just worth flagging that pending rows are now
  interactively clickable in a way they weren't before.
- No tests added or modified. UI-only, snapshot-free repo for this
  component, so this matches existing convention — not blocking.

## Verdict rationale

`merge-as-is`: fixes a real UX/safety bug, the diff is purely a deletion
of `!pending()` guards that were never correct, and the only side effects
(subtitle clickability during pending) are arguably also desirable. No
need for changes before merge.
