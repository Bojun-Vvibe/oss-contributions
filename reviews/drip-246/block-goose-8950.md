# PR #8950 — feat: goose2 compact settings modal headers

- Repo: block/goose
- Author: kalvinnchau
- Head: `896c7852715eb553e24adfde04e14001d3d9050f`
- URL: https://github.com/block/goose/pull/8950
- Verdict: **merge-after-nits**

## What lands

15-file +443/-388 refactor of the goose2 settings modal. Three things
happen at once:

1. **Decompose `SettingsModal.tsx` into per-section files.** Three new
   files are added (`AboutSettings.tsx`, `ChatsSettings.tsx`,
   `ProjectsSettings.tsx`) and `SettingsModal.tsx` shrinks from ~245
   inline-section lines to dispatching to those components.
2. **Compact and standardize per-section headers** via a shared
   `SettingsPage.tsx` wrapper.
3. **Move the Escape-to-close keyboard handler** from a `<div onKeyDown>`
   on the modal root to a document-level event listener at
   `SettingsModal.tsx:707-713`.

## Specific findings

- `ui/goose2/src/features/settings/ui/SettingsModal.tsx:686-692` removes
  six pieces of state (`isTransitioning`, `archivedProjects`,
  `archivedChats`, `loadingArchived`, `loadingArchivedChats`,
  `deletingProject`) plus the matching effects at `:701-718` and the
  `handleRestoreProject` / `handleRestoreChat` / `handleDelete`
  callbacks at `:720-742`. All of that logic moves into the new
  `ProjectsSettings.tsx` and `ChatsSettings.tsx` (line 388 / 158
  respectively). Decomposition is correct and isolates archive-CRUD
  responsibility to the section that owns the data.
- `:707-713` replaces the prior `<div role="dialog" onKeyDown=...>`
  pattern (which required the dialog div to be focused) with a
  document-level `addEventListener("keydown", ...)`. Also adds proper
  `aria-modal="true"` and `aria-label={activeSectionLabel}` at `:766-767`.
  Both are accessibility wins. The cleanup at `:752` returns the
  `removeEventListener` from the effect, so no leak.
- `:759-760` derives `activeSectionLabel` via `navItems.find(item =>
  item.id === activeSection)?.label ?? t("title")`. Correct fallback to
  the i18n `title` if no nav item matches.
- `:744-750` removes the `setIsTransitioning(true)` /
  `setTimeout(() => setIsTransitioning(false), 150)` and the
  `transition-all duration-400 ease-out` opacity/translate fade. This
  collapses ~30 lines of styling but also eliminates the entrance fade
  between sections — that's a UX regression unless intentional. The PR
  body should call this out; otherwise reviewers will assume the fade
  was lost by accident.
- `:763` adds a `biome-ignore lint/a11y/useKeyWithClickEvents` directive
  with rationale "Escape is handled by the document listener while the
  backdrop only handles pointer dismissal." Correct rationale; future
  contributors won't be able to "fix" the lint by re-adding onKeyDown.
- `:783-784` swaps the content container's `relative flex-1 overflow-y-auto`
  for `relative flex min-w-0 flex-1 flex-col` and pushes `min-h-0 flex-1
  overflow-y-auto` down to an inner `<div>` at `:816`. The `min-h-0
  flex-1` pattern is the correct Tailwind/flexbox idiom for "let me
  scroll inside a flex container" — same idiom kalvinnchau used in #8946
  one drip back. Consistent.
- The close button moves from `z-10` to `z-30` at `:792`. Worth
  confirming this doesn't conflict with any other floating UI inside
  sections (e.g. dropdowns); without seeing all sections rendered the
  z-index bump is plausible but easy to break.

## Risks

- The opacity/translate transition removal at `:744-750` is a visible UX
  change. If unintentional, the PR title ("compact settings modal
  headers") doesn't capture it — the title should probably be "refactor
  settings modal: per-section components + compact headers + remove
  inter-section transition."
- The document-level keydown listener at `:707-713` will fire even when
  the modal isn't focused. Effect dependency is `[onClose]` so if `onClose`
  changes identity the listener is correctly re-bound. But if multiple
  modals are stacked (settings + a confirmation dialog inside settings),
  Escape will fire `onClose` even when the user expects it to dismiss the
  inner dialog. Given the AlertDialog imports were removed at `:644-654`
  (no inner dialogs in settings any more), this is currently safe — but
  fragile if a future PR re-adds a confirmation dialog.
- 15-file diff with no test changes shown, except the new
  `SettingsPage.test.tsx` at line 1101. Worth confirming the existing
  modal-level tests (Escape-closes, click-backdrop-closes,
  navigate-between-sections) still pass with the new dispatch shape.

## Verdict

**merge-after-nits**. Decomposition is right, the a11y improvements
(`aria-modal`, `aria-label`, document-level keydown) are wins, and the
flex/overflow idiom matches the established pattern from #8946. Two
follow-ups before merge: (1) call out the inter-section transition
removal in the PR description so it's an intentional UX change not
collateral, (2) add or confirm a regression test for "Escape closes the
modal even when focus is inside a section component" since the keydown
handler moved off the dialog root.
