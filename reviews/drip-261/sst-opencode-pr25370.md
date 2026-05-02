# sst/opencode PR #25370 — fix(ui,app): allow context tooltip on touch and add mobile context tab

- **Head SHA:** `9069ac0eb5e40d80b7263c55d41c4c19aa27704b`
- **Files:** 2 (`packages/app/src/pages/session.tsx`, `packages/ui/src/components/tooltip.tsx`)
- **LOC:** +63 / −45

## Observations

- `tooltip.tsx:129-132` — gating `onPointerDownCapture` on `event.pointerType !== "touch"` is the right fix for the "tap dismisses before tap activates" anti-pattern on iOS Safari. But there is no companion long-press / focus path added, so a touch user can never *open* the tooltip. The PR title says "allow context tooltip on touch" — the change actually *suppresses* the touch-driven arm path. Verify this is intentional (i.e., the new mobile context tab makes the tooltip unnecessary on touch) and not a regression of the original feature.
- `session.tsx:516` — widening `mobileTab` to `"session" | "changes" | "context"` and the matching `!w-1/2` → `!w-1/3` rewrite (`session.tsx:1805,1813,1821`) are mechanical and safe. Tab labels rely on `language.t("session.tab.context")` — confirm the i18n key exists in every locale bundle, otherwise mobile users on non-default locales will see the raw key.
- `session.tsx:1848-1894` — the `<Show when={!isDesktop() && store.mobileTab === "context"} fallback={<MessageTimeline .../>}>` indents the entire MessageTimeline prop block by ~2 levels. Reviewable but creates a ~50-line noisy diff. A small extraction (`<MobileContext />` vs `<MobileTimeline />`) would have made the intent clearer; non-blocking.
- No tests added for the new tab routing; given the surrounding file has none either, this matches local convention.

## Verdict: `merge-after-nits`

- Confirm the touch-tooltip behavior change is intentional and that `session.tab.context` is localized.
- Consider extracting the timeline branch to keep `Page()` readable.
