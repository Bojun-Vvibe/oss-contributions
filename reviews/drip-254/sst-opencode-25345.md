# sst/opencode #25345 — fix(opencode): fix infinite selection loop when hovering in TUI menus

- **Repo:** sst/opencode
- **PR:** https://github.com/sst/opencode/pull/25345
- **HEAD SHA:** `127acb24918c06a743e487950940268e6221767c`
- **Author:** thoughtlesslabs
- **Verdict:** `merge-after-nits`

## What the diff does

Closes #25310 — the "infinite ping-pong selection loop when moving
the mouse rapidly over TUI dialog/autocomplete menus" symptom — by
breaking the reactive feedback loop at three independent points in
two files. The bug shape: the dialog-select effect at `dialog-
select.tsx:146-158` was a `createEffect(on([filter, current],
...))` with a `setTimeout`-deferred body that called `moveTo(...)`
unconditionally; mouse hover wrote `current`, the effect fired,
the effect called `moveTo`, `moveTo` wrote `selected`, which the
mouse-position calculator read back as the new highlighted row
index, which triggered another `current` write, which scheduled
another `setTimeout`, ad infinitum.

Files of note:

- `packages/opencode/src/cli/cmd/tui/ui/dialog-select.tsx:146-158`
  — the load-bearing change. The effect now captures the
  *previous* `(filter, current)` tuple via the `prevInput`
  parameter of `on(...)`, and inside the deferred timeout body:
  - At `:151`: `if (!isDeepEqual(props.current, current)) return`
    — the abort check. By the time the `setTimeout` callback
    runs, the live `props.current` may have moved on (mouse moved
    again before the timeout fired); if so, *this* timeout's job
    is stale and shouldn't write to `store.selected`. This is the
    primitive that breaks the runaway feedback loop — only the
    most-recent `(filter, current)` pair gets to write, every
    earlier pending timeout aborts.
  - At `:154`: `if (filter !== prevFilter && filter.length > 0)`
    — only call `moveTo(0)` when the filter actually *changed*
    AND is non-empty. Prior code called `moveTo(0)` on every
    filter-effect-fire even when filter hadn't changed (current
    moved instead).
  - At `:157-158`: `else if (!isDeepEqual(current, prevCurrent)
    && current)` — only call `moveTo(currentIndex, true)` when
    `current` actually changed (and is truthy). Prior code re-
    centered the viewport on every effect fire even when current
    hadn't moved.
- `packages/opencode/src/cli/cmd/tui/ui/dialog-select.tsx:175-184`
  — `moveTo` itself gains:
  - `if (store.selected === next) return` — the early-return
    on no-op. Stops the function from writing to `selected` when
    the new index matches the current one (which would
    redundantly trigger reactive subscribers).
  - `if (store.input === "mouse") return` — the mouse-input
    bypass on the auto-scroll-to-center branch. Prior code did
    the `scroll.getChildren().find(...)` viewport-centering
    behavior unconditionally; now mouse-driven selection changes
    don't aggressively re-center the viewport (which would yank
    the row the user is hovering over out of position, causing
    the next mouse-coordinate read to map to a different row,
    causing another selection change). Keyboard-driven selection
    (input === "keyboard") still gets the centering, which is the
    right asymmetry — keyboard users want the highlighted row in
    view, mouse users have already placed the cursor where they
    want it.
- `packages/opencode/src/cli/cmd/tui/component/prompt/autocomplete.
  tsx:482-487` — same two early-returns added to autocomplete's
  `moveTo`: `if (store.selected === next) return` (no-op skip)
  and `if (store.input === "mouse") return` (mouse-input scroll-
  bypass). Smaller diff because autocomplete's wrapping `effect`
  doesn't have the same re-fire problem dialog-select had.

## Why it's right

The diagnosis is correct: the infinite loop is a classic reactive
feedback cycle, and the fix targets the three independent ways the
cycle was being driven (stale-timeout writes, no-op writes,
viewport recentering on mouse hover). Each of the three guards
breaks the cycle at a different point:

1. **`isDeepEqual(props.current, current) → return` at `:151`**
   is the load-bearing primitive — without it, every queued
   `setTimeout` (one per mouse-position-driven `current` change)
   would eventually run and write `selected`, and rapid mouse
   movement over the menu produces dozens of queued timeouts in
   rapid succession. By aborting all but the most-recent, the
   chain of pending callbacks collapses to one effective write.

2. **The `filter !== prevFilter` and `isDeepEqual(current,
   prevCurrent)` guards at `:154,157`** narrow the effect's
   actual writes to "only when the input that changed is the
   relevant input." Prior code reset the index to 0 on every
   filter-effect-fire (including when filter hadn't changed),
   which on its own wasn't the loop-driver but contributed to
   the unnecessary write traffic.

3. **`store.input === "mouse" → skip auto-center` at `:181`** is
   the cleanest break: mouse hover that lands on a row should
   not yank the viewport (which would move the row out from
   under the cursor and create a new mouse-coordinate-to-row
   mapping that triggers another selection change). Keeping
   keyboard-driven scroll-centering working unchanged is the
   right asymmetry.

4. **`store.selected === next → return` at `:175`** is the
   universal no-op skip — even outside the loop scenario, this
   is the right reactive-system hygiene (don't write the same
   value you already have).

The fix preserves all the working behavior:
- Keyboard-driven selection still auto-centers the viewport
  (tested via the absence of mouse-input-mode in the keyboard
  path).
- Filter-driven selection still resets to index 0 (gated on
  filter actually changing).
- `current`-driven selection still finds and centers the matching
  row (gated on current actually changing).

The dual application of the same guards in both `dialog-select.tsx`
and `autocomplete.tsx` is correct because both components have the
same `moveTo`-on-input pattern, and both have the mouse-input mode
distinguishable from keyboard via the same `store.input` field.

The PR description names the bug shape and links #25310; the
attached video evidence (two recordings) is the right validation
shape for "this is a visual/interaction bug that's hard to assert
in unit tests."

## Nits

1. **Two manually-tested screen recordings, no automated test.**
   The cycle bug is hard to express in a TUI-rendering unit test,
   but a property-style test using a mock `store` and a recorded
   sequence of `setStore("input", "mouse")` + `setStore("current",
   ...)` events asserting bounded write count to `selected` would
   catch a future regression. The minimum useful unit test:
   "given 100 rapid `current` changes within one tick, `setStore
   ("selected", ...)` is called at most once."

2. **`prevFilter`/`prevCurrent` from `prevInput` parameter at
   `:147-149`** — the `on(...)` callback signature has
   `prevInput: [string, T] | undefined` (undefined on the first
   fire), and the destructure `const prevFilter = prevInput ?
   prevInput[0] : undefined` handles the first-fire case. Worth a
   one-line comment about why the first fire's `prevInput` is
   undefined ("first effect fire has no prior input to compare
   against; both filter !== prevFilter and isDeepEqual(current,
   prevCurrent) read as 'changed' on first run, which is the
   right semantics — initial position should be applied"), so a
   future reader doesn't try to "fix" the undefined handling.

3. **`isDeepEqual` is presumably from `@solid-primitives/deep` or
   similar** — not imported in the visible diff hunk. Confirm
   it's already imported at the top of the file (the
   non-displayed import block must include it since the existing
   `findIndex((opt) => isDeepEqual(opt.value, current))` at the
   prior line uses it).

4. **`store.input === "mouse"` magic string** — the input mode is
   a string literal in two places (`dialog-select.tsx:181`,
   `autocomplete.tsx:486`). A typed `InputMode` enum/union with
   constants would prevent typos in future call sites. Mechanical
   refactor.

5. **`moveTo`'s `center` boolean parameter at `:175`** is now
   unused when `store.input === "mouse"` because the function
   early-returns before the centering branch — fine, but worth
   a comment that the `center` parameter is keyboard-input-only
   so future callers don't pass `center: true` for a mouse
   interaction expecting it to do something.

6. **Comment at `:149-150`** ("Abort if the state changed while
   waiting in the timeout queue / This breaks the infinite ping-
   pong reactivity loop when moving the mouse quickly") is good.
   Mirror it in autocomplete.tsx near the `moveTo` early-returns
   so a future maintainer of that file sees the same intent named
   when they trace the bug.

7. **`if (filter.length > 0)` becomes `if (filter !== prevFilter
   && filter.length > 0)` at `:154`** — the `&&`-conjunction is
   correct, but the order matters for short-circuit performance:
   `filter !== prevFilter` (cheap pointer comparison) before
   `filter.length > 0` (also cheap, but slightly more work for
   the JS engine). Current order is correct and idiomatic. No
   action needed.

## Verdict rationale

Right diagnosis (reactive feedback loop driven by stale-timeout
writes + viewport-recenter-on-mouse-hover triggering new
mouse-coordinate-to-row mappings), right load-bearing primitive
(`isDeepEqual(props.current, current) → return` aborts stale
timeouts so only the most-recent write effective), right asymmetry
(keyboard-driven scroll-centering preserved, mouse-driven scroll-
centering bypassed), right secondary guards (no-op skip on equal,
filter-changed gate on filter-reset path), correctly applied to
both `dialog-select.tsx` and `autocomplete.tsx` with the same
pattern. Two manually-recorded videos validate the symptom is
gone. Wants a property-style test for write-count bound under
rapid input, comment on the first-fire `prevInput === undefined`
semantics, typed input-mode constants instead of magic strings,
mirrored "abort stale timeout" comment in autocomplete.

`merge-after-nits`
