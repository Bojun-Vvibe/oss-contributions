# sst/opencode PR #25426 — fix(app): confirm workspace deletion on Enter

- URL: https://github.com/sst/opencode/pull/25426
- Head SHA: `850d8c4fbc981f9f26ffd152f8c05f95deefa923`
- Files touched: 3 (+42 / −9)
- Verdict: **merge-after-nits**

## Summary
Wires Enter into the workspace-delete confirmation dialog. Adds a pure helper
`workspaceDeleteKeyAction(event, status)` exporting a 3-state result
(`ignore | block | confirm`) and a window-level keydown listener registered on
mount of the dialog. The helper has dedicated unit tests.

## Cited concerns

1. `packages/app/src/pages/layout.tsx:1626-1646` (post-patch):
   `makeEventListener(window, "keydown", …, { capture: true })`. Capture phase
   on `window` will swallow Enter for *any* focused control while this dialog
   is mounted, including inputs inside other portals/popovers that happen to
   render concurrently. Worth verifying there is no nested form (e.g. a
   "type the workspace name to confirm" field) where the user expects Enter
   to commit text rather than fire delete. If such a control is added later,
   this listener will quietly hijack it.

2. `helpers.test.ts:221-227`: tests cover `Escape→ignore`, `Enter+loading→block`,
   `Enter+repeat→block`, `Enter+ready→confirm`. Good coverage of the gate, but
   no test asserts that `event.preventDefault()` / `stopPropagation()` are
   actually invoked on `confirm` vs `block` paths — those side effects are part
   of the contract callers depend on. A `vi.spyOn(event, "preventDefault")`
   assertion would lock that in.

3. `layout.tsx:1647-1649`: `handleDelete()` is now declared *before* the
   `onMount` block (the diff hoists it). Make sure none of the other call
   sites referenced earlier in the file changed import order — the move looks
   purely lexical, but the previous declaration was inside the same closure
   so hoisting is fine. Worth a second pair of eyes.

4. Accessibility: confirm the dialog also traps focus and that the native
   `<dialog>` Escape cancel still wins over this listener. The patch does not
   touch focus management.

## Verdict
**merge-after-nits** — solid little fix with a pure helper + tests. Tighten the
capture-phase concern with a comment or scope it to the dialog root, and the
patch is ready.
