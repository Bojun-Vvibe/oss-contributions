# block/goose #9018 — keep settings open during window drag

- **Head SHA:** `fb429659db87defd4713ab0c81d36b8903832344`
- **Author:** @morgmart
- **Verdict:** `merge-as-is`
- **Files:** +116 / −8 across `ui/goose2/src/features/settings/ui/SettingsModal.tsx` (+51/−8) and `ui/goose2/src/features/settings/ui/__tests__/SettingsModal.test.tsx` (+65, new file)

## Rationale

Tight, well-tested fix for a real Tauri-app UX bug: the previous backdrop wired `onClick={onClose}` directly on the modal root container (`SettingsModal.tsx:105`, pre-PR), so any click landing on the backdrop dismissed settings — including the *click-up* terminating a window-drag gesture initiated on the backdrop. Tauri lets users drag the window from any region marked `data-tauri-drag-region`, so a user mousing-down on the backdrop, dragging the window 50px, and releasing would trigger the `click` handler at release-time and unexpectedly return to Home. This PR makes that gesture work the way users expect.

**The fix at `SettingsModal.tsx:97-124` is correct in three pieces:**

1. **Splits the backdrop into a dedicated DOM element** (lines 137-144 post-PR) instead of overloading the modal root container. The new `<div data-testid="settings-backdrop" data-tauri-drag-region className="absolute inset-0 bg-background/80 backdrop-blur-sm" onPointerDown={…} onClick={…} />` carries the drag region and pointer handlers in one place; the modal panel sibling sits at `relative z-10` (line 148) above it. Clean separation of concerns — the backdrop is for dismissal-or-drag, the panel is for content.

2. **`handleBackdropPointerDown` (lines 97-105)** records `{x, y}` in a ref *only when the pointer-down lands on the backdrop itself* (`event.target !== event.currentTarget` early-out). That target check matters because event bubbling from a child would otherwise capture pointer-down coordinates from the panel-edge children and confuse the threshold check.

3. **`handleBackdropClick` (lines 107-122)** computes `Math.hypot(deltaX, deltaY)` against the recorded pointer-down position and calls `onClose()` only if `moved <= BACKDROP_CLOSE_DRAG_THRESHOLD` (=4 pixels, defined at line 45). Threshold of 4px is the standard "click vs drag" tolerance — matches platform conventions for mouse/trackpad jitter on click-release. Falls back to `onClose()` if `pointerDown` is null (covers e.g. keyboard-driven click activations where there was no pointer-down to record).

The ref is *cleared after every consult* (line 113: `backdropPointerDownRef.current = null;`) so a stale value can't leak between gestures. The early-out at line 109 (`event.target !== event.currentTarget`) ensures clicks on inner panel children that bubble up don't trigger the close (though those bubble-paths are also stopped by the panel sibling no longer needing its old `onClick={(e) => e.stopPropagation()}` — line 113 of the pre-PR file, removed in this diff).

**Tests at `__tests__/SettingsModal.test.tsx`:**

- Lines 53-60: backdrop click with pointerDown and click at the same coordinates → `onClose` called once. Pins the "click still closes" requirement.
- Lines 62-72: pointerDown at `(20, 20)` followed by click at `(44, 20)` (Δ=24px, well above 4px threshold) → `onClose` not called. Pins the "drag-like motion preserves modal" requirement.

Both tests use `screen.getByTestId("settings-backdrop")` against the new `data-testid` added at line 140 — that's the right way to target the backdrop without coupling the test to the (intentionally complex) class string. The 9 module mocks at lines 5-41 stub out every settings sub-section so the test loads the modal in isolation — overhead is justified because settings imports a lot.

**Minor observations, none blocking:**

- The threshold constant `BACKDROP_CLOSE_DRAG_THRESHOLD = 4` is module-scoped. If a future fix needs to tune it (e.g. for touch displays where finger jitter is larger), one constant is easy to bump. Could be tested at the boundary (move=3px → close, move=5px → no close), but the current two tests adequately pin both sides of the decision.
- The test file uses `fireEvent` rather than `userEvent`. For a click-vs-drag distinction this is fine — the threshold logic only reads `clientX/clientY`, not the pointer trajectory between events, so `fireEvent` faithfully reproduces the production code path.
- `handleBackdropPointerDown` doesn't capture `pointerId` for `setPointerCapture`-style guarantees. That's the right call: pointer capture would interfere with Tauri's drag-region behavior, which needs the OS to take over the pointer mid-gesture.

The PR body is honest about the limitation it doesn't address ("Full goose2 pre-push/typecheck remains blocked by existing unrelated SDK definition mismatches on `main`") — those mismatches predate this PR and aren't introduced by it. The two targeted Vitest + Biome runs the author cites are the right validation surface for a behavior change in one component.

Ship it.
