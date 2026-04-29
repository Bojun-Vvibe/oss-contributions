# Review: google-gemini/gemini-cli#26139 — fix(cli): fix bugs from stale closures in FooterConfigDialog

- **Author**: psinha40898
- **Head SHA**: `c5782842215daf5968585d62d45775127d45151f`
- **Diff**: +180 / −37 across 4 files
- **Fixes**: #26138

## What it does

Closes two related stale-closure bugs in the `/footer` config TUI:

- **Bug 1**: After unchecking a default item (e.g. "workspace") and exiting, reopening `/footer` and pressing "Reset to default" requires *two* presses to take effect. First press is observed to do nothing; second press works.
- **Bug 2**: After reset, the highlighted (active) row stays at the *old* first index instead of moving to the new first index of the restored default order.

Root cause is the asynchronous nature of `setSetting` from `useSettingsStore`: `handleResetToDefaults` was calling `setSetting(SettingScope.User, 'ui.footer.items', undefined)` and *then* immediately calling `resolveFooterState(settings.merged)` to build the new state — but `settings.merged` in the closure was the value at render time, not the value after the in-flight `setSetting` write had propagated, so the "new" state was actually the old state. The second click finally saw the propagated update.

The fix has three substantive shapes:

1. **`FooterConfigDialog.tsx`** — pulls `showLabels` into the local reducer state instead of reading it from `settings.merged.ui.footer.showLabels` on every render (which had its own stale-closure variant for the labels toggle), adds a `'TOGGLE_LABELS'` action, renames `'SET_STATE'` (partial-merge semantics) to `'RESET'` (full-replace semantics, payload-typed `FooterConfigState`), and rewrites `handleResetToDefaults` (`FooterConfigDialog.tsx:172-194`) to:
   - **Not call `setSetting` at all during reset.** Instead, build a synthetic `defaultFooterSettings` object (`...settings.merged, ui: { ...settings.merged.ui, footer: { ...settings.merged.ui.footer, items: undefined } }`), pass it to `resolveFooterState` to derive the new state purely locally, dispatch `RESET` with that state, and call `setFocusKey(defaultState.orderedIds[0])` to move the highlight.
   - The actual write to `setSetting` is deferred until `handleSave`, which now also writes `showLabels` if it changed: `if (state.showLabels !== currentShowLabels) setSetting(SettingScope.User, 'ui.footer.showLabels', state.showLabels)` (line 165-168).

2. **`useSelectionList.ts:310-353`** — reorders two `useEffect` blocks so that the items-changed initialization effect runs *before* the `focusKey`-changed effect. Header comments at lines 313-316 and 339-341 spell out: "This effect must run BEFORE the focusKey effect below so that when both items and focusKey change simultaneously (e.g., after a reset), focusKey's SET_ACTIVE_INDEX runs last and wins." React fires effects in declaration order on a render where deps changed, so this physical ordering is the load-bearing fix for Bug 2.

3. **Tests**:
   - `FooterConfigDialog.test.tsx:264-336` adds two ink-renderer tests:
     - `reset to default works on first press` — initial state has `workspace` unchecked, navigates 12 down-arrows to "Reset to default", presses Enter once, and `waitFor` asserts `[✓] workspace` (the default state restored).
     - `active index moves to first item after reset` — same setup, asserts the active marker `> [✓] workspace` is on the new first item.
   - `useSelectionList.test.tsx:834-859` (`should respect focusKey even when items change (regression test for reset bug)`) — starts with `focusKey: 'C'` at index 2, `rerender`s with a new items list where `C` is now at index 3 (a new `NEW` item inserted at the front), and asserts `result.current.activeIndex === 3` (focusKey followed C, didn't stick to old index 2).

## Concerns

1. **`handleResetToDefaults` is now a *purely-local* state mutation that doesn't actually clear the persisted setting.** Pre-PR, reset wrote `setSetting(... 'ui.footer.items', undefined)`; post-PR, reset only updates the in-memory dialog state and waits for `handleSave` to persist on dialog close. That means: (a) if the user clicks "Reset to default" then closes the dialog with Esc *without* saving, nothing is persisted — but did the old behavior write on reset-button-press alone? If so, this is a subtle UX regression where reset is no longer an immediate-commit action. (b) Conversely, if the user resets, then re-customizes, then saves, the customizations are saved (correct). The PR description doesn't address (a). Worth either a comment in the dialog explaining the deferred-commit semantics, or a "you have unsaved changes" guard on Esc.

2. **`getDefaultValue('ui.footer.showLabels')`** is called from `handleResetToDefaults` (line 191) but the test for `showLabels` reset isn't shown in the diff. Both new tests are about `items` reset only. A third test that flips `showLabels` to `false`, hits "Reset to default", and asserts `showLabels` returns to its default (`true`) would close the matrix.

3. **The reducer rename `SET_STATE` → `RESET` removes the partial-update capability.** Pre-PR, `SET_STATE: { type: 'SET_STATE'; payload: Partial<FooterConfigState> }` accepted a partial; post-PR, `RESET: { type: 'RESET'; payload: FooterConfigState }` requires a full state. That's the right shape for the use case (reset semantics are full-replace), but if any other code path in the codebase dispatched `SET_STATE` with a partial, it'll fail typecheck. The diff doesn't show that grep — `rg "SET_STATE" packages/cli/src/ui/components/FooterConfigDialog` would confirm.

4. **The two-`useEffect`-must-run-in-this-order pattern is fragile.** React doesn't guarantee effect ordering across files, and a future refactor that hoists one of these effects into a custom hook breaks the invariant silently. The header comments help, but a single `useEffect(() => { ... initialize then handle focusKey ... }, [items, focusKey])` would make the sequencing explicit. Not a blocker — the existing two-effect shape is consistent with how the rest of `useSelectionList` is structured — but worth a `// react-effect-ordering-load-bearing` marker comment too.

5. **Manual validation matrix is sparse.** PR template shows MacOS+Linux `npm run` only — no Windows, no Docker, no `npx`. For a TUI keypress-driven dialog, that's not exhaustive but it is in line with prior PRs to this file.

## Verdict

**merge-after-nits** — correct identification of a stale-closure bug pair with the right structural fix (decouple "compute new state" from "persist new state") and a regression-pinning test in `useSelectionList`. The deferred-commit semantics for reset (Concern 1) is the only behavioral question worth answering before merge; the other items are tightening.

