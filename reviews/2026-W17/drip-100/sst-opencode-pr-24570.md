# sst/opencode#24570 — fix(app): keep question dock buttons visible on mobile when options have long descriptions

- **Author**: ysm-dev
- **Size**: +5 / -4 (1 file)
- **Issue**: #23730

## Summary

The question dock's `measure()` callback in `packages/app/src/pages/session/composer/session-question-dock.tsx:122-135` looked at `scroller.firstElementChild` and only set `--question-prompt-max-height` if that first child carried the `sticky` class. A prior refactor wrapped the sticky session title inside a `setContentRef` div, so the `firstElementChild`-and-`.sticky` check stopped matching, the CSS var was never set, and the dock fell back to `100dvh` — clipping the Dismiss/Submit buttons below the iPhone viewport on long-description questions.

The fix swaps the brittle "first child + class probe" for a `scroller.querySelector("[data-session-title]")` lookup with `scroller.getBoundingClientRect().top` as the untitled-session fallback, then bails to `removeProperty(...)` only when the scroller itself is missing.

## Diff inspection

```diff
-    const head = scroller instanceof HTMLElement ? scroller.firstElementChild : undefined
-    const top =
-      head instanceof HTMLElement && head.classList.contains("sticky") ? head.getBoundingClientRect().bottom : 0
-    if (!top) {
+    if (!(scroller instanceof HTMLElement)) {
       root.style.removeProperty("--question-prompt-max-height")
       return
     }

+    const sticky = scroller.querySelector("[data-session-title]")
+    const top =
+      sticky instanceof HTMLElement ? sticky.getBoundingClientRect().bottom : scroller.getBoundingClientRect().top
```

Behavior matrix:
| Scenario | Old behavior | New behavior |
| --- | --- | --- |
| Titled session, sticky child first | computes `head.bottom` ✓ | computes `sticky.bottom` ✓ |
| Titled session, sticky child wrapped | falls to `top=0` → bail → `100dvh` ✗ | finds `[data-session-title]` ✓ |
| Untitled session (no title) | bail → `100dvh` | uses `scroller.top` (sensible default) |
| Scroller absent | bail | bail |

The wrapped-title regression is the actual reported bug; the untitled-session improvement is a small bonus.

## Strengths

- Selector-based lookup is structurally robust to wrapper additions (the prior code coupled to DOM order).
- `data-session-title` is a stable hook intent-named for this use, less likely to drift than a layout class.
- Fallback to `scroller.top` for untitled sessions is the right default — previously untitled sessions also lost the max-height clamp.
- Five-line patch with screenshots before/after on iPhone 14 viewport.

## Concerns

1. **No regression test.** The function is DOM-coupled and the file likely has no jsdom unit tests today, but a single Vitest snapshot exercising the three branches (sticky present / absent / scroller missing) would lock down the contract for the next refactor.
2. **The `[data-session-title]` attribute itself isn't shown in the diff.** Need to confirm `packages/app/src/pages/session/index.tsx` (or wherever the title renders) actually emits `data-session-title` on the sticky element — otherwise the new branch silently degrades to the untitled fallback. (PR description and screenshots confirm runtime behavior, but a one-line grep should be in the maintainer's checklist.)
3. **Scroller selector `.scroll-view__viewport` is also class-coupled** — same brittleness class as the original bug. Out of scope for this fix, but worth a follow-up to add a `data-scroll-viewport` hook so the same regression can't recur on the scroller itself.

## Verdict

**merge-after-nits** — confirm `[data-session-title]` is actually emitted by the sticky title element (one grep), then merge. Test coverage and the scroller-selector hardening can be follow-ups.
