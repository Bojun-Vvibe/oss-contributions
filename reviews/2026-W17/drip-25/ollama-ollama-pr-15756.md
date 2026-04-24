# ollama/ollama PR #15756 — app/ui: fix sidebar animating open on initial load

- **Repo:** ollama/ollama
- **PR:** [#15756](https://github.com/ollama/ollama/pull/15756)
- **Head SHA:** `439603db7d73fe6c7b01ca3182934a809ed34edd`
- **Author:** anishesg (Anish Kataria)
- **Size:** +42 / −5 across 3 files
- **Reviewer:** Bojun (drip-25)

## Summary

Fixes #12954: when the desktop app loads with a persisted
`SidebarOpen: true`, the sidebar visibly animates from closed →
open instead of appearing in its persisted state immediately.

Root cause is the canonical "default-while-loading + late
async resolve = layout flash" pattern:
- `useSettings` defaults `sidebarOpen` to `false` while the
  settings API request is in flight.
- Once the response arrives with `SidebarOpen: true`, the width
  class flips from `w-0` to `w-64`.
- The CSS `transition-[width]` rule kicks in and produces a
  visible slide animation on every cold load.

Fix suppresses sidebar-related CSS transitions until *after*
the initial settings have been fetched and a frame has been
painted, then re-enables transitions so subsequent user-driven
toggles still animate normally.

## Key changes

### `app/ui/app/src/hooks/useSettings.ts` (+3 / −2)

Adds a flag indicating "initial settings load complete". Used
by the layout to gate transition suppression. Standard "is the
async data ready?" boolean — pattern is fine. Worth confirming
the flag also covers the *failure* case (if the settings API
returns an error, do transitions ever get re-enabled?). If the
flag is keyed on success-only, the sidebar would stay
transitionless forever after a failed initial fetch.

### `app/ui/app/src/components/layout/layout.tsx` (+4 / −3)

The actual fix — a `useEffect` that, on the load-complete
signal, schedules a `requestAnimationFrame` callback to re-enable
transitions. The `rAF` is the load-bearing detail: it ensures
the browser has *painted* the correct sidebar width at least
once before transitions are re-enabled, so the next paint sees
the same width and doesn't animate.

The standard alternative (a CSS class flip via `useState` +
`useEffect`) would be vulnerable to the React batching/commit
window producing the same visible transition. Using `rAF` to
push the re-enable into the next paint frame is correct.

### `app/ui/app/src/utils/sidebarAnimation.test.ts` (+35 / 0)

New unit-test file covering the transition-suppression logic.
Doesn't exercise the actual rAF + paint cycle (hard to do in
JSDOM), but does pin the suppress/release state machine
behaviour so a future refactor can't silently regress it.

## Concerns

1. **Failure-mode coverage.**
   What happens if the settings API call fails or hangs? The PR
   description and the diff don't explicitly cover this case.
   Two failure modes worth thinking through:
   - **Hang**: the load-complete flag never flips → sidebar
     stays in its default-`false` state with transitions
     suppressed → if the user manually toggles it, does it
     animate? If transitions are still suppressed, no — and
     the user would see an instant snap, which is worse UX
     than the original bug. A safe fix: add a timeout (200–
     500ms) after which transitions get re-enabled regardless.
   - **Error**: settings API returns 500 → does the load-
     complete flag still flip? If it's wired to "settings
     resolved successfully" only, the same staying-suppressed
     issue applies. Should flip on *any* terminal state of the
     fetch (success, error, abort).

2. **The fix is layout-specific; siblings could regress similarly.**
   The "default-while-loading + late resolve" pattern almost
   certainly affects other persisted UI state too (e.g. theme,
   density, panel collapse states). Worth a follow-up issue to
   audit `useSettings` for other persisted-vs-default booleans
   and apply the same suppress-until-painted pattern, or
   factor it into a reusable `useSuppressTransitionsUntilFirstPaint`
   hook. As-is, the next persisted UI flag added to settings
   will likely cause the same kind of visible flash.

3. **`requestAnimationFrame` cleanup not visible in the diff
   summary.**
   If the `rAF` callback is scheduled in a `useEffect` that
   doesn't return a cleanup function calling
   `cancelAnimationFrame`, an unmount during the load window
   will fire the callback against a stale ref. Standard React
   hygiene issue, easy to miss. Worth confirming the
   `useEffect` returns the cancel — and adding a unit test for
   the unmount-during-loading case.

4. **Transition suppression mechanism choice.**
   PR body says "suppresses all sidebar-related CSS transitions
   until after the initial settings have been fetched and a
   frame has been painted". The two common ways to implement
   this are:
   - Conditionally add a `no-transition` CSS class to the
     sidebar while suppressed (then remove it after rAF).
   - Inline `style={{ transition: 'none' }}` while suppressed.

   Both are fine; the class-based approach plays better with
   `prefers-reduced-motion` user-agent overrides. Worth
   confirming the chosen approach respects the user's motion-
   reduction preference (i.e. doesn't *enable* transitions on a
   user who has them disabled).

## Verdict

`merge-after-nits` — the fix is correctly diagnosed (default-
while-loading + late async resolve triggers the CSS transition
on cold load) and correctly implemented (suppress until a
post-load `rAF` confirms the paint). Three pre-merge asks:
confirm the load-complete flag flips on *any* terminal fetch
state (not just success), confirm the `useEffect` cancels its
`rAF` on unmount, and confirm the chosen suppression mechanism
respects `prefers-reduced-motion`. One follow-up worth filing:
audit `useSettings` for other persisted booleans that could
have the same flash-on-cold-load shape, or extract the
suppress-until-first-paint logic into a reusable hook.

## What I learned

The "default-while-loading + late async resolve = layout
flash" pattern is one of the most common UI bugs in any app
with persisted UI state, and it's easy to miss in dev because
the dev environment usually has hot module reloading that
bypasses the cold-load window. The defensive shape is to treat
the *first paint after async data resolution* as a
synchronization barrier — suppress transitions across that
barrier, then re-enable. `requestAnimationFrame` is the right
tool for "after the next paint" because it runs after the
browser commits the layout but before the next frame is
shown. The other lesson: any `default-while-loading` boolean
in a UI hook is a latent flash waiting to happen, and the
right fix is at the *transition-enabling* layer (suppress the
animation), not the *value-providing* layer (pretend the
default value never existed). The latter is tempting but
breaks down as soon as a third consumer reads the same hook.
