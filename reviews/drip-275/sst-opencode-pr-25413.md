# Review: sst/opencode #25413

- **PR:** sst/opencode#25413 — `fix: allow context tooltip on touch devices and add mobile context tab`
- **Head SHA:** `53998d2ae3f870f2797737ad8d031d201dfb1ebd`
- **Files:** 4 (+71/-50)
- **Verdict:** `merge-after-nits`

## Observations

1. **Tooltip touch fix is the right pattern** — `packages/ui/src/components/tooltip.tsx` skipping `arm()` when `pointerType === "touch"` matches what Kobalte's own `TooltipTrigger` does to avoid double-handling touch events. The companion change to allow `justClickedTrigger` to open (not just close) is correct: previously the click-suppression flag was symmetric and locked the tooltip out of the open state on touch.

2. **Mobile tab arithmetic is correct but i18n-fragile** — switching `!w-1/2` → `!w-1/3` plus `!px-3` in `session.tsx:1802-1830` divides space evenly, and the PR body itself acknowledges Spanish "4 Archivos Cambiados" will truncate. Worth a follow-up issue (already noted by author) but acceptable for this fix.

3. **`SessionContextTab` rendering branch wraps the entire `<MessageTimeline>` in `<Show fallback={...}>`** at `session.tsx:~1848-1894`. This means on every `mobileTab` flip the timeline component fully unmounts/remounts, which loses scroll position, virtualization state, `historyWindow` cache, etc. A `Switch`/`Match` with three siblings (or keeping the timeline mounted with `display: none`) would preserve user state when toggling Session ↔ Context. Nit-level but the scroll loss will be felt.

4. **`onClick` prop default in `session-context-usage.tsx:115`** uses `props.onClick ?? openContext` — fine for this case, but `openContext` references a closure with `args` (sessionID, etc.) that aren't captured by the override path. Worth confirming the mobile-tab switch handler doesn't need any of that state (it appears it doesn't, since the context tab reads from session context independently).

## Recommendation

Land the tooltip fix as-is; consider the timeline-remount nit in a follow-up.
