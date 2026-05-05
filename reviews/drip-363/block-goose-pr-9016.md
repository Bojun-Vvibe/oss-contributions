# block/goose #9016 — make collapsed sidebar search actionable

- Head SHA: `169d521f34c86e2053f8d855c5b92b814137f9bf`
- Author: morgmart
- Diff: +67 / −15 across 4 files (`Sidebar.tsx`, `Sidebar.test.tsx`, en + es locale jsons)

## Verdict: `merge-after-nits`

## What it does

Fixes a real "looks interactive but does nothing" UX bug in the goose2 sidebar. Previously, when the sidebar was collapsed, the search affordance was rendered as a `<div>` containing only an `IconSearch` — no click handler, no role, no focus target. Users would click it expecting search to open and get nothing.

The fix:
1. **Splits the rendering** between collapsed and expanded states (`Sidebar.tsx:342-379`). Collapsed state now renders a `<Button variant="ghost" size="icon-sm">` with `onClick={activateCollapsedSearch}`, `aria-label`/`title` from `t("actions.search")`, and tabbable focus styles. Expanded state keeps the existing `<div>` + `<input>` layout but drops the `transition-all` class and the `{!collapsed && ...}` conditional gate (since the branching is now at the top level).
2. **`activateCollapsedSearch`** (`Sidebar.tsx:277-280`) sets `focusSearchOnExpand=true` then calls `onCollapse()`. Pure intent-recording; no DOM manipulation here.
3. **A focus-on-expand effect** (`Sidebar.tsx:127-135`) fires when `collapsed` flips to `false` while `focusSearchOnExpand` is true. Uses `requestAnimationFrame` to defer focus until after the layout pass that follows the collapse-state change, then resets the flag. Cleanup function cancels the rAF on unmount/dependency change.
4. **i18n keys added** for both `en` and `es` (`actions.search: "Search chats"` / `"Buscar chats"`).
5. **A regression test** (`Sidebar.test.tsx:135-163`) renders the sidebar collapsed, clicks the new button, asserts `onCollapse` was called once, then re-renders with `collapsed={false}` and `waitFor`s the placeholder input to receive focus.

## What's good

1. **The focus-deferral pattern is correct.** `requestAnimationFrame` runs the callback before the next paint but *after* the layout pass triggered by React re-rendering with the new `collapsed=false` prop. That's exactly the right hook point — using `setTimeout(0)` would race against React's commit phase, and calling `.focus()` synchronously in the click handler would target the input that doesn't exist yet (the input only mounts when `collapsed` is false).
2. **The cleanup function is correct.** `cancelAnimationFrame(frame)` runs on either unmount OR when `collapsed`/`focusSearchOnExpand` change before the rAF fires — preventing a focus call after unmount and preventing two queued focus calls if the user spam-clicks.
3. **The `focusSearchOnExpand` flag is reset inside the rAF callback,** so a single click → single focus. If `setFocusSearchOnExpand(false)` were called outside the rAF, the next render could re-enter the effect and queue a duplicate rAF.
4. **i18n key is added consistently to both locales.** The PR description explicitly notes "so the sidebar locale files stay complete" — and indeed the test asserts the English copy `"Search chats"` (line 148) so a missing en key would have flagged at test time.
5. **The collapsed `<Button>` uses `variant="ghost" size="icon-sm"`** matching the rest of the collapsed sidebar's button styling (the diff shows the same shape used elsewhere in the file). Not introducing a snowflake button style.
6. **Test exercises the actual user flow,** not the implementation. The pattern of "render collapsed → click → re-render with collapsed=false → waitFor focus" mirrors what the parent component would do in production. Good shape.
7. **Author was honest about scope.** PR body explicitly notes that "Full goose2 pre-push/typecheck remains blocked by existing unrelated SDK definition mismatches on `main`" with three named missing exports. That's what scoped validation looks like.

## Nits worth raising before merge

### 1. `requestAnimationFrame` runs at-most-once per frame; if `focusSearchOnExpand` flips back on before the previous frame's reset, you can lose a focus

Walk through: user clicks collapsed-search → `setFocusSearchOnExpand(true)` + `onCollapse()`. Effect fires, schedules rAF. Before the rAF fires, parent re-mounts the sidebar with a new `key` (e.g. workspace switch). The cleanup function `cancelAnimationFrame(frame)` runs and the new mount starts with `focusSearchOnExpand=false`. Now the user is in a state where they expected search to focus but the focus call was cancelled. Probably fine in practice — the parent re-mount is the "weirder" event — but worth a one-line comment explaining the cleanup trade-off.

### 2. Test asserts `getByPlaceholderText("Search chats...")` but the placeholder string isn't visible in the diff

`Sidebar.test.tsx:158` does `expect(screen.getByPlaceholderText("Search chats...")).toHaveFocus()`. The placeholder must be coming from the existing `<input placeholder={t("actions.searchPlaceholder")}>` or similar — but that key doesn't appear in the locale diff. If a future i18n cleanup pass renames `searchPlaceholder` to something else, this test breaks silently (it'll throw `Unable to find element with placeholder text`). Asserting against `screen.getByRole("textbox", { name: ... })` or capturing the input by `data-testid` would be more refactor-resilient. Not a blocker.

### 3. `aria-label` and `title` are both set to `t("actions.search")` — same string

Screen readers prefer `aria-label`; sighted users see the `title` tooltip. Identical text is the conventional safe choice here. No change needed; flagging only because some accessibility audits flag duplicate-label patterns. Current code is correct per WAI-ARIA practices for icon-only buttons.

### 4. Spanish translation uses `"Buscar chats"` while English uses `"Search chats"`

Both fine. Note that the English collapsed-button label at the test level (line 148) is `"Search chats"` — exact match — so any future tone-pass that changes "Search chats" to "Search conversations" needs to update both the locale file AND the test in the same PR. A `screen.getByRole("button", { name: t("actions.search") })` pattern would decouple the test from the literal copy, but that requires the test to import or mock `t`, which is a heavier change.

### 5. `mb-3 h-10 w-full rounded-md` on the collapsed button is a layout opinion baked into the button

The previous `<div>` had `p-3` for the collapsed case; the new `<Button>` uses `h-10 w-full`. These produce visually similar (but not identical) target sizes. If the design system has a canonical "collapsed-sidebar icon button" sizing (e.g. `size="icon"`), this PR is a hand-rolled near-match. Worth confirming with design before this becomes the precedent for other collapsed-sidebar affordances.

## Why not request-changes

The bug is real, the fix is well-reasoned (rAF deferral + cleanup + flag-reset-inside-callback all done correctly), the test exercises the user flow, and i18n is complete in both supported locales. The five nits above are robustness/style polish, not correctness.

## Citations

- `ui/goose2/src/features/sidebar/ui/Sidebar.tsx:90` — `focusSearchOnExpand` state
- `ui/goose2/src/features/sidebar/ui/Sidebar.tsx:127-135` — focus-on-expand effect with rAF
- `ui/goose2/src/features/sidebar/ui/Sidebar.tsx:277-280` — `activateCollapsedSearch` intent-recorder
- `ui/goose2/src/features/sidebar/ui/Sidebar.tsx:342-379` — collapsed-vs-expanded render split
- `ui/goose2/src/features/sidebar/ui/__tests__/Sidebar.test.tsx:135-163` — regression test
- `ui/goose2/src/shared/i18n/locales/{en,es}/sidebar.json` — `actions.search` key in both locales
