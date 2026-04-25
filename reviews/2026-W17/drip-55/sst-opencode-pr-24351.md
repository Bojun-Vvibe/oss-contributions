# sst/opencode PR #24351 — keep question dock buttons visible on mobile

@cff3f253 · base `main` · +9/-9 · author `ajaytravel`

## Summary
Closes #23730 and #22439. On mobile, the session question dock's next/dismiss buttons were being clipped below the viewport when the dock had long content. Root cause: `measure()` was reading the offset from `.scroll-view__viewport`'s sticky child first, and silently bailing out (`removeProperty`) when there wasn't a sticky header — leaving no max-height set and letting content overflow.

## What changed
- `packages/app/src/pages/session/composer/session-question-dock.tsx:121-138` — reorders `measure()`:
  - moves the `dock` lookup to the top so the early-return triggers before any DOM probing
  - replaces the "scroller's first sticky child" lookup with a more direct `[data-session-title]` query, falling back to the scroller's top, and finally to `0`:
    ```ts
    const top =
      header instanceof HTMLElement
        ? header.getBoundingClientRect().bottom
        : scroller instanceof HTMLElement
          ? scroller.getBoundingClientRect().top
          : 0
    ```
  - drops the early `if (!top) { removeProperty(...); return }` branch — `top: 0` now flows through to the max-height calc, which is the right behavior on mobile where there's no sticky header.

## Key observations
- The PR body's diagnosis is accurate: the previous code's "no sticky head ⇒ skip max-height entirely" branch was the bug. Removing that early return and falling through with `top: 0` is the right fix.
- Switching from a structural query (`scroller.firstElementChild + classList.contains("sticky")`) to a semantic data attribute (`[data-session-title]`) is a meaningful improvement — that lookup was brittle to DOM-order changes elsewhere in the scroll view.
- Dock-lookup hoist is a small but real correctness win: previously a missing dock would still pay the cost of the sticky-header probe.
- The fallback chain is three levels deep (`header → scroller → 0`); add a one-line comment explaining the precedence, otherwise the next refactor will collapse it.

## Risks/nits
- Confirm there's exactly one `[data-session-title]` element on the page; multiple matches would silently pick the first and that may or may not be the right one.
- `boundingClientRect().bottom` of the header includes margin/border but not transforms — if the title bar is animated in via `transform`, the measurement could lag during transitions. Probably fine for this use case but worth noting.
- No test in the diff window — visual/layout fixes are notoriously hard to unit-test, but a Playwright assertion that the next/dismiss buttons are within viewport bounds at 375px width would lock this in.

**Verdict: merge-after-nits**
