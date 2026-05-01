# sst/opencode #25345 — fix(opencode): fix infinite selection loop when hovering in TUI menus

- URL: https://github.com/sst/opencode/pull/25345
- Head SHA: `127acb24918c06a743e487950940268e6221767c`
- Files: `packages/opencode/src/cli/cmd/tui/component/prompt/autocomplete.tsx` (+2/-0), `packages/opencode/src/cli/cmd/tui/ui/dialog-select.tsx` (+12/-3)
- Closes #25310

## Context / problem

Mouse-driven hover over TUI autocomplete and dialog-select menus triggered an infinite ping-pong: hover fires `moveTo(idx)` → `setStore("selected", idx)` → reactive effect on `[filter, current]` schedules a `setTimeout` → timeout calls `moveTo` again on the (possibly stale) value → state thrash. The recursive write also kept auto-scrolling the viewport to re-center the selection while the user was hovering, making the menu jump under the cursor.

## Design analysis

Three-line shape, one fix per failure mode:

1. **Idempotent `moveTo`** at `autocomplete.tsx:482` and `dialog-select.tsx:169`:
   ```ts
   if (store.selected === next) return
   ```
   Cuts the recursion at the source — the store no longer fires its `selected` subscribers when the value is unchanged, so the reactive effect that schedules the `setTimeout` never re-arms.
2. **Mouse-input scroll suppression** at `autocomplete.tsx:485` and `dialog-select.tsx:178`:
   ```ts
   if (store.input === "mouse") return
   ```
   Auto-centering on mouse-driven selection is wrong UX (the viewport shouldn't jump under the cursor); keyboard-driven `moveTo` keeps the existing scroll-into-view behavior.
3. **Stale-effect abort** at `dialog-select.tsx:148-152`:
   ```ts
   on([() => store.filter, () => props.current], ([filter, current], prevInput) => {
     const prevFilter = prevInput ? prevInput[0] : undefined
     const prevCurrent = prevInput ? prevInput[1] : undefined
     setTimeout(() => {
       if (!isDeepEqual(props.current, current)) return
       if (filter !== prevFilter && filter.length > 0) { moveTo(0, true) }
       else if (!isDeepEqual(current, prevCurrent) && current) { ... }
     })
   })
   ```
   The `prevInput` capture turns the previously-unconditional `setTimeout` body into a "did this dimension actually change since last fire" check, so a `current`-only change no longer fires the filter-reset arm and a `filter`-only change no longer fires the current-restore arm. Combined with the `props.current !== current` deep-equal guard, the timeout body is now a no-op when the world has moved on while it was queued — exactly the breakable link in the ping-pong.

## Risks

- The `store.input === "mouse"` predicate assumes `input` is reset to a non-mouse value when keyboard navigation resumes; if there's no such reset (e.g. mouse hover sets it but tab/arrow doesn't clear it), keyboard scroll-into-view stays broken until the next non-mouse input event. Worth a one-line verification by the author.
- The `prevInput` parameter is the SolidJS `on(..., callback)` "previous values" tuple, which is `undefined` on first run — the `prevInput ? prevInput[0] : undefined` guard handles that correctly, so first-mount filter-with-content still triggers `moveTo(0, true)` (good).

## Suggestions

- Add a `// no-op on stale tick` comment at `dialog-select.tsx:147` so the next contributor doesn't "simplify" the abort guard back into a dead loop.
- A regression test would be hard to write at the TUI integration level, but a unit test asserting `moveTo(N)` followed by `moveTo(N)` only writes the store once would lock the idempotence at the API boundary.

## Verdict

`merge-after-nits` — three correct fixes for three distinct failure modes (recursion at the writer, scroll-side-effect on the wrong input, stale-tick re-application on the wrong dimension), AI-assisted but the author has clearly understood and verified each one. Wants the small comment + a unit test for the idempotence guard.
