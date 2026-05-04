# block/goose #8946 — fix goose2 window minimum sizing

- Link: https://github.com/block/goose/pull/8946
- Head SHA: `ff4106c5390de1c198fd36a807fa297af6d8702b`
- State: MERGED 2026-05-01, +31/-27 across 2 files

## Summary
Two related fixes for the new "goose2" Tauri shell:

1. **Window-level**: declares a `minWidth: 800, minHeight: 600` on the
   Tauri window so the user cannot drag it to a size where the UI breaks.
2. **Modal-level**: the Settings modal previously hardcoded `h-[600px]`
   and `w-full max-w-3xl`. On small windows / certain zoom levels the
   modal would overflow the viewport and the sidebar nav couldn't scroll
   to reveal hidden items. The PR replaces the height with
   `h-[min(600px,calc(100vh-4rem))]` and the width with
   `w-[calc(100vw-2rem)] max-w-3xl`, then makes the sidebar a scrollable
   flex region (`min-h-0 flex-1 overflow-y-auto`) so its nav buttons
   reach via scroll when there are too many to fit.

## Specific references
- `ui/goose2/src-tauri/tauri.conf.json:22-23` — adds
  `"minWidth": 800, "minHeight": 600` to the window config.
- `ui/goose2/src/features/settings/ui/SettingsModal.tsx:157` — modal
  outer container's classNames: height becomes
  `h-[min(600px,calc(100vh-4rem))]`, width becomes
  `w-[calc(100vw-2rem)] max-w-3xl`.
- `ui/goose2/src/features/settings/ui/SettingsModal.tsx:165` — sidebar
  flex container gains `min-h-0 flex-shrink-0` so it correctly shrinks
  inside the bounded outer height.
- `ui/goose2/src/features/settings/ui/SettingsModal.tsx:179-205` — `<nav>`
  becomes `min-h-0 flex-1 overflow-y-auto px-2 pb-3` and the buttons get
  wrapped in an inner `<div>` so the scroll container and the flex
  layout don't conflict.

## Notes
- The Tailwind arbitrary values (`h-[min(600px,calc(100vh-4rem))]`,
  `w-[calc(100vw-2rem)]`) are the exact right idiom for "cap at design
  size, but never overflow viewport with a small margin". The 2 rem /
  4 rem margins look like the standard breathing room around a modal.
- Wrapping the buttons in an extra `<div>` inside `<nav>` is needed
  because making `<nav>` itself the scroll container while it's also a
  flex column would constrain the buttons to share the scroll height.
  The comment in the diff doesn't explain this, but the structure is
  correct.
- `min-h-0` on a flex-column child is the well-known Tailwind workaround
  for the "default `min-height: auto` prevents overflow:auto from
  scrolling" pitfall — applied correctly to both the sidebar column and
  the inner nav.
- No tests, but visual-only regression in a Tauri shell is hard to test
  cheaply. Manual smoke-test on small windows is sufficient.

## Verdict
`merge-as-is`
