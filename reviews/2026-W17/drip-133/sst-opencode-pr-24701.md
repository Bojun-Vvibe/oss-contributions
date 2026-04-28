# sst/opencode #24701 — fix: use continuous scroll layout for share page on desktop

**Verdict:** `merge-after-nits`

- Repo: `sst/opencode`
- PR: https://github.com/sst/opencode/pull/24701
- Head SHA: `c4b937d9`
- Author: workdocyeye
- Diff: +3 / -64, single file `packages/enterprise/src/routes/share/[shareID].tsx`
- Closes: #18567 (related: #6576, #20916)

## What it does

Deletes the desktop-only `MessageNav` sidebar layout from the public share
page and lets the mobile-style continuous-scroll `turns()` view render at all
viewport widths. The motivating bug: the sidebar showed every entry as the
generic label "New Message" so users couldn't tell which tick was which, and
the hover tooltip vanished as soon as the cursor moved toward the sidebar.

## Design analysis

The change is the right shape. It is a pure deletion: the only added line is
the blank line replacing the removed `MessageNav` import at
`packages/enterprise/src/routes/share/[shareID].tsx:21`. The body of the
change drops a 64-line `<div classList={{...md:flex / lg:flex...}}>` block
(`:285-345` in the pre-image) that contained the `<Show when={messages().length > 1}>`
+ `MessageNav` + single-`SessionTurn` arrangement, plus the `Show
when={diffs().length > 0}` `SessionReview` companion column.

Then it strips the `md:hidden / lg:hidden` classList conditions from the
mobile branches so the `<Tabs>` and the `turns()` `<Switch><Match>` both
become unconditional:

```
-                                <Tabs classList={{ "md:hidden": wide(), "lg:hidden": !wide() }}>
+                                <Tabs>
...
-                                <div classList={{ "!overflow-hidden": true, "md:hidden": wide(), "lg:hidden": !wide() }}>
+                                <div class="!overflow-hidden">
```

That is the correct minimum touch — `wide()` was only ever used to pick
between the now-deleted desktop block and the mobile branches, so once the
desktop block is gone the conditional collapses cleanly.

The `turns()` function itself is untouched. The author's verification claim
("the same function already working in production for mobile viewports") is
literally true from the diff: there are no behavior changes to message
rendering, diff rendering, `SessionTurn`, `SessionReview`, or any data
plumbing — desktop just stops getting a different code path.

## Risks

1. **`wide()` may now be dead code on this route.** The diff removes both
   call sites of `wide()` inside this file's render tree. If the signal isn't
   referenced elsewhere in the file (the diff context doesn't show a
   declaration), this leaves a dangling derivation. Worth a one-line follow-up
   to delete `wide()` and its source signal so the file doesn't accumulate
   unused state.
2. **Loss of cross-message navigation on large screens.** The deleted
   `MessageNav` was the only random-access affordance — continuous scroll
   makes long shared sessions a long scroll on desktop too. The bug being
   fixed (every entry labeled "New Message") is real and the sidebar is worse
   than no sidebar in its current state, so deletion is defensible, but a
   future PR should bring back keyboard / hash-jump navigation to specific
   message IDs once the labels are real.
3. **`SessionReview` (diffs panel) is now mobile-tab-only.** Pre-PR, desktop
   had a side-by-side messages + diffs layout via the deleted right column;
   post-PR, desktop users with diffs see them only via the `<Tabs>` switcher,
   which is a mobile-tabs UX on a 27" monitor. Acceptable as an intermediate
   state but worth a follow-up.

## Suggestions

- Delete the `wide()` signal and its declaration in the same PR so the file
  doesn't ship dead state.
- Add a screenshot of the post-PR desktop layout to the PR body — the
  pre-image is in #18567 but the after-image isn't in this PR, and reviewers
  on the enterprise package would benefit from seeing the diffs-tab fallback.
- Consider whether the `select-text flex flex-col flex-1 min-h-0` wrapper at
  `:284` still needs the `min-h-0` after the inner desktop block went away;
  it's harmless but easy to verify-or-prune.

## What I learned

This is the "we built a separate desktop layout, it diverged, and the
divergence shipped a bug" pattern that justifies "one layout, responsive
where forced." Deleting the divergent path and letting the mobile path
render at all widths is almost always cheaper than fixing the desktop path
in isolation, and the diff stat (+3 / -64) is the strongest single argument
for the change. The right follow-up is bringing back the *function* the
deleted sidebar provided (jump to message N) without bringing back the
divergent *layout*, e.g. via in-flow message anchors.
