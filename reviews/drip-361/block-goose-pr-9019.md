# block/goose #9019 — fix goose2 small-window chat and settings layouts

- **Head SHA:** `270600151176cdf64acb9d0a35b02477af5f2673`
- **Author:** @morgmart
- **Verdict:** `merge-after-nits`
- **Files:** +73 / −22 across `ui/goose2/src/features/chat/ui/{AgentModelPicker,ChatContextPanel,ChatInput,ChatInputSelector,ChatInputToolbar}.tsx` and `ui/goose2/src/features/settings/ui/SettingsModal.tsx`

## Rationale

Six-file responsive-layout fix for the goose2 desktop UI when run at narrow widths. The diff splits cleanly into three independent UI behaviors: (a) the context panel switches from a width-consuming layout sibling to a right-edge overlay below 900px viewport width (`ChatContextPanel.tsx:17` — new `CP_COMPACT_QUERY = "(max-width: 900px)"` plus a `useEffect` at lines 44-56 that subscribes via `matchMedia.addEventListener("change", …)` and unsubscribes correctly in cleanup); (b) the input toolbar wraps its picker controls and re-anchors the action group with `flex-wrap gap-y-2` + `ml-auto` when `isCompact` is true (`ChatInputToolbar.tsx:212-224,294`); (c) the settings modal switches to a stacked top-nav + bottom-content layout with horizontal nav scroll below 640px viewport width (`SettingsModal.tsx:104-148` — Tailwind `max-[640px]:` arbitrary-breakpoint variants throughout).

**What's done well:**

- The `matchMedia` subscription in `ChatContextPanel.tsx:44-56` correctly guards against `!window.matchMedia` for non-browser test envs, sets initial state from `mediaQuery.matches` synchronously in the same effect, and uses `addEventListener("change", …)` (not the deprecated `addListener`). Cleanup removes the listener — no leak.
- The compact-overlay positioning at `ChatContextPanel.tsx:74-87` uses `absolute bottom-3 right-3 top-12 z-10 w-[min(340px,calc(100%-1.5rem))]` which clamps the panel to a sensible max width (340px) while preserving 12px padding from the right edge — that's the correct way to stop a 340px panel from overhanging a 320px viewport.
- The `shadow-modal` class added at `ChatContextPanel.tsx:91-94` only when `isCompactViewport` is on is the right call — overlay needs visual elevation to look like it's floating, but the docked variant doesn't need it because it sits inside the parent's border.
- `AgentModelPicker.tsx:105` adds `max-w-full` alongside the existing `min-w-0` — together those two utilities are exactly the "shrink-wrap inside a flex parent" idiom needed to let the picker truncate its label without overflowing the toolbar. Same idiom appears at `ChatInputSelector.tsx:74` (`triggerVariant === "toolbar" && "max-w-40"`) — both are the right fix for the right symptom.
- `ChatInput.tsx:373-379,385` reduces outer + inner padding at `<sm` breakpoints (`px-2 pb-3 pt-0 sm:px-4 sm:pb-6` outer; `px-3 sm:px-4` inner) — preserves existing desktop spacing exactly while making narrow viewports breathable.
- `SettingsModal.tsx:105-148` is the most invasive sub-change but is purely additive — every existing class is preserved, with `max-[640px]:` variants appended. The stacked layout (`max-[640px]:flex-col`), the sidebar that becomes a top nav (`max-[640px]:max-h-32 max-[640px]:w-full max-[640px]:border-b max-[640px]:border-r-0`), the horizontal scroll (`max-[640px]:overflow-x-auto max-[640px]:overflow-y-hidden`), and the row-flex nav items (`max-[640px]:min-w-max max-[640px]:flex-row`) are internally consistent.

**Concerns:**

1. **No test coverage.** The PR body says "validated through the goose2 build, check, typecheck, Tauri checks, and Vitest suite" but adds no new Vitest. For a layout fix that introduces a `matchMedia` subscription and viewport-conditional rendering, a small RTL test that mocks `window.matchMedia` and asserts the overlay class kicks in below 900px would be cheap insurance. Companion PR #9018 (settings drag) does add tests in the same file area, so the pattern exists.

2. **900px and 640px are duplicated breakpoints.** The 900px lives as a string constant in `ChatContextPanel.tsx:17` (`CP_COMPACT_QUERY`) and the 640px lives inline as Tailwind's `max-[640px]:` arbitrary-value variant repeated ~10 times in `SettingsModal.tsx`. That's two different sources of truth for "what counts as narrow", and they don't match (900 vs 640). At least add a code comment explaining the rationale ("context panel needs to overlay sooner because it's adding 340px; settings modal can stay docked longer because it's already responsive width") — otherwise a future contributor will assume one is wrong and "fix" them to match.

3. **`max-w-32` / `max-w-56` truncation widths in `AgentModelPicker.tsx`** (existing code at line 107, not modified by this PR but exposed by the new `max-w-full` parent constraint) — confirm 32 (=128px) is enough for the shortest meaningful model label like "qwen-coder" before truncation kicks in. With the new `max-w-full` parent, the inner `max-w-32` becomes the actual truncation point and might be tighter than before.

4. **No screenshots / demo.** PR body acknowledges this. For a pure layout fix the maintainer should validate visually before merge — a single before/after pair at 320×600 (iPhone SE-class) and 800×600 would confirm the breakpoints behave as intended.

The fix is correct and the implementation is idiomatic Tailwind + React. Land after a Vitest covering the matchMedia subscription and one comment reconciling the 900/640 split. The truncation widths and screenshots are nice-to-haves.
