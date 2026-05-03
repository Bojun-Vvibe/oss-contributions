# sst/opencode PR #25554 — fix(titlebar): keep "new chat" icon visible on smaller viewports

- Author: Wdits
- Head SHA: `5962f56e6810025265a08dda06b93857873897b5`
- Diff: +32 / -32, single file `packages/app/src/components/titlebar.tsx`
- Closes #25551

## Observations

1. **`titlebar.tsx:222-258` lifts the `new-session` button out of the `hidden xl:flex` wrapper** and into a sibling `<Show when={params.dir}>` block that renders unconditionally at all breakpoints. The `+32 / -32` line balance is mostly indentation: the inner `<div class="transition-opacity" ...>` and `TooltipKeybind` content are byte-identical to the previous version — this really is a structural move, not a behavior rewrite.
2. **The `aria-hidden` toggle and `tabIndex={-1}` on `Button` when `layout.sidebar.opened()`** are preserved, so the previous a11y contract (button is removed from the focus ring while the sidebar's own new-session affordance is the active one) still holds. Good — that's the subtle bit you don't want to lose in a layout move.
3. **The remaining `hidden xl:flex` wrapper at the new `titlebar.tsx:259+`** still gates the *other* sibling controls (the cluster after the new-session button). Worth a line in the PR description confirming which controls are intentionally still `xl:`-gated, since the diff title only mentions the one icon — a reviewer eyeballing it might assume the whole cluster is now always-visible.
4. **No test coverage added.** The component has no obvious snapshot/visual harness in this repo, so that's acceptable, but a one-line note in the PR linking to a manual breakpoint sweep (375 / 768 / 1024 / 1280 / 1920) would close the loop on "smaller viewports" — the PR text says "verified" but doesn't list widths.

## Verdict: `merge-after-nits`

Surgical move, a11y wiring preserved, intent matches the linked issue. Pre-merge: have the author confirm in the PR body which sibling controls remain `xl:`-gated so future readers don't assume the whole cluster is always-visible, and ideally name the breakpoints they tested at.
