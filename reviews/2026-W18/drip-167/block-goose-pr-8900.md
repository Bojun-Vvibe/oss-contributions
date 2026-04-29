# block/goose PR #8900 — fix: prevent model picker from becoming disabled during loading

- Repo: block/goose
- PR: https://github.com/block/goose/pull/8900
- Head SHA: `eac93b5dc3e1049c53991201a73ca68a508fac5f`
- Author: matt2e (Matt Toohey)
- Size: +23 / −39, 1 file

## Context

The agent model picker at `ui/goose2/src/features/chat/ui/AgentModelPicker.tsx`
had three startup-time UX papercuts: (1) the trigger button showed
"Loading…" text even when a previously-selected provider/model label was
already in state, producing visual flicker on every cold start; (2) the
button itself was disabled the entire time `loading` was true, so a user
couldn't open the picker until the inventory fetch resolved; (3) every
single picker open re-fetched the provider inventory, despite that fetch
already being kicked off at app startup; (4) when the dropdown was opened
mid-load, the model list area showed only "Loading models…" instead of
hinting at what's currently selected. This PR addresses all four.

## What the diff actually does

Net −16 lines on a single file. The interesting changes:

- **Removes the per-open inventory refetch** at the old
  `useEffect(..., [open, mergeInventoryEntries])` block (deleted, plus
  the now-unused imports of `getProviderInventory` and
  `useProviderInventoryStore`). This is the highest-leverage piece —
  every picker open was firing a network round-trip that the startup
  flow had already done.
- **Tightens the disabled gate** at `AgentModelPicker.tsx:366`:
  `disabled={loading}` becomes `disabled={loading && !selectedAgentLabel}`.
  Reading top-down: if a label is already known (i.e. provider was
  previously selected and is still in state), the button stays
  clickable even while inventory is reloading.
- **Tightens the trigger label** at `AgentModelPicker.tsx:373-375`:
  `loading ? t("toolbar.loading") : (triggerModelLabel ?? selectedAgentLabel)`
  becomes `triggerModelLabel ?? selectedAgentLabel ?? (loading ?
  t("toolbar.loading") : null)`. Same outcome on a fresh install (shows
  Loading), but on a warm start the label appears immediately and never
  flashes through the loading state.
- **Replaces the bare loading row** in the dropdown (old: a centered
  spinner + "Loading models…" string) with a richer treatment that, when
  a `currentModelName || currentModelId` is already known, shows that
  model as a `selected disabled` `PickerItem` with a small inline
  spinner. The fallback (no current model known) still shows the bare
  spinner-plus-string.

## Risks

The change is small and self-contained but a few things are worth a
second look:

1. **Removing the per-open inventory fetch removes a recovery path.**
   Today, a user whose startup-time inventory fetch failed for
   transient network reasons gets a second chance every time they open
   the picker — silent, but recovers. After this PR, the only way
   inventory ever loads is the startup fetch; a transient failure leaves
   the user with a permanently-empty picker until they restart Goose.
   Worth either (a) keeping the open-time fetch but gating it on
   `inventory.length === 0` so it only fires when there's nothing to
   show, or (b) adding a small "retry" affordance in the empty-state
   row.

2. **`triggerModelLabel ?? selectedAgentLabel ?? (loading ? ... : null)`
   renders `null` as button text when nothing is known and we're not
   loading.** The old code at minimum always rendered *something* (the
   loading string). A button with empty text + an icon may render as a
   visually-broken trigger in the no-data, not-loading window between a
   failed inventory fetch and the user noticing. Worth a fallback like
   `t("toolbar.selectModel")` for the empty-not-loading case.

3. **The new dropdown loading state shows the current model as
   `selected` even though no actual selection is happening.** That's
   consistent UX, but the `disabled` flag on the `PickerItem` means the
   user can't click it to confirm the current choice while loading. If
   a user wants to "yes, keep this one, I just want to dismiss the
   picker", they can't — the only escape is clicking outside. Verify
   that closing the popover does the right thing in that case (no-op,
   not "clear selection").

4. **`PickerItem`'s `selected` + `disabled` combo styling.** Many
   component libraries don't render a sensible visual for the
   intersection of these two states (often the disabled gray overrides
   the selected highlight, and the user sees a dimmed row that looks
   like "this option is unavailable"). Worth a quick screenshot or
   visual regression to confirm the row reads as "this is your current
   model" and not "this option is unselectable".

5. **No tests in this PR.** The PR body has a manual test plan but no
   automated coverage. For a UX-only change in a React component this is
   defensible, but a testing-library snapshot of the trigger render in
   the four states `(loading, hasLabel) ∈ {true, false}²` would lock in
   the loading-flicker fix.

## Suggestions

- Either reintroduce a guarded `if (inventory.length === 0)` open-time
  fetch, or add a manual retry affordance in the empty state.
- Provide an `t("toolbar.selectModel")` fallback for the empty +
  not-loading button label so the trigger never renders as empty text.
- Verify close-popover behavior with the disabled-current-model row.
- Add a small testing-library test pinning the trigger render across
  the four loading/label states.

## Verdict

**merge-after-nits** — net cleanup, correct intent, and the four
papercuts addressed are real. The recovery-path removal in (1) is the
most important nit; the rest are polish.
