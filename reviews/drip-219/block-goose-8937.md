---
pr-url: https://github.com/block/goose/pull/8937
sha: b652a346d8b3
verdict: merge-after-nits
---

# fix(goose2): avoid transform-rasterized dialog text

UI fix swapping the dialog/alert-dialog positioner shape from `grid place-items-center p-4` (CSS centering via grid placement, which forces the Radix `Content` element through a CSS transform that some platforms rasterize at non-integer subpixel offsets, blurring text) to a three-row template `grid grid-rows-[minmax(1rem,1fr)_auto_minmax(1rem,1fr)] justify-items-center overflow-y-auto px-4` with the `Content` pinned to `row-start-2`. Applied symmetrically across `ui/goose2/src/shared/ui/alert-dialog.tsx:51-58` and `ui/goose2/src/shared/ui/dialog.tsx:57-64`. The Radix-managed `data-[state=open]:zoom-in-95` / `data-[state=closed]:zoom-out-95` transforms are also dropped from both `Content` className strings — those zoom animations were the *direct* cause of the rasterization (they leave the element in a `transform: scale(...)` state mid-animation and some compositors snapshot during that window), so removing them is the load-bearing change; the row-template re-layout is the alignment-preserving structural change that lets the removal land cleanly.

A nice secondary win: the new `overflow-y-auto px-4` on the positioner means dialogs taller than the viewport now scroll inside the modal rather than overflowing off-screen — which is a correctness improvement that was hidden behind the cosmetic motivation. The `data-slot="dialog-positioner"` / `alert-dialog-positioner` attributes added at `:53` / `:51` are also useful: they give downstream consumers a stable hook for variant styling without monkey-patching the inner Content node.

Nits: (1) dropping the zoom animation is a behavior change — users who liked the open/close motion will notice the absence; a `prefers-reduced-motion` -respecting alternative (e.g. opacity-only fade, no transform) would split the difference rather than removing motion entirely; (2) `max-w-lg` replaces `max-w-[calc(100vw-2rem)] ... sm:max-w-lg` — on viewports narrower than `lg` (32rem default) this changes the dialog's responsive cap; verify on mobile breakpoints that nothing clips; (3) no visual-regression test or screenshot harness is added, so the rasterization fix is asserted-by-claim — fine for a UI polish PR but means the next compositor regression won't be caught in CI.

## what I learned
CSS transforms (`scale`, `translate` for centering, etc.) interact with GPU rasterization in ways that are platform-specific and animation-state-specific — when text in a transformed element looks slightly fuzzy on one OS but not another, the fix is almost always "remove the transform, re-layout via grid/flex placement," not "tweak the transform value." The grid-row-template centering pattern in this PR is the right reusable shape.
