# PR #25309 — fix(opencode): fix infinite selection loop when hovering in TUI menus

- Repo: sst/opencode
- Author: thoughtlesslabs
- Head: `127acb24918c06a743e487950940268e6221767c`
- URL: https://github.com/sst/opencode/pull/25309
- Verdict: **merge-as-is**

## What lands

Two-file, +14/-3 fix targeting a feedback loop where mouse-hover in
`Autocomplete` and `DialogSelect` would re-trigger the auto-scroll path,
which moved the highlight under the cursor again, which re-fired the hover
handler, etc. The fix breaks the loop at three independent gates and is the
right minimum surface.

## Specific findings

- `packages/opencode/src/cli/cmd/tui/component/prompt/autocomplete.tsx:484`
  early-returns from `moveTo(next)` when `store.selected === next`. This is
  the cheap idempotence guard that should always have been there — without
  it, even a no-op hover write to `store.selected` triggers Solid's
  reactivity chain.
- `autocomplete.tsx:486` adds `if (store.input === "mouse") return` to skip
  the scrollTop/scrollBottom adjustment when the input source is the mouse.
  Correct: the user's cursor has already chosen which row to look at, so
  auto-scrolling to recenter the row is exactly what re-fires hover at a
  new row and creates the ping-pong.
- `dialog-select.tsx:146-156` is the more interesting half. The
  `createEffect(on([...filter, current], ([filter, current], prevInput) =>
  ...))` now captures `prevFilter`/`prevCurrent` and inside the `setTimeout`
  callback aborts via `if (!isDeepEqual(props.current, current)) return`
  when the prop has changed since the timer was queued. This is the
  textbook fix for "stale work scheduled into a microtask queue" — the
  comment at `:148-150` calls it out explicitly.
- The new `if (filter !== prevFilter && filter.length > 0)` and `else if
  (!isDeepEqual(current, prevCurrent) && current)` arms (`:153,155`)
  correctly distinguish "filter actually changed" from "current actually
  changed" instead of unconditionally re-running both branches on any tick
  of the effect. Without these guards, a mouse-hover that updated `current`
  via `onMove?.(option)` would re-trigger the filter branch and reset
  selection to 0.
- `dialog-select.tsx:175-181` mirrors the autocomplete fix one-for-one
  (idempotence guard + mouse-input skip on the scroll path). Good
  consistency — both menu surfaces share the same hover-loop shape.

## Risks

- The `if (store.input === "mouse") return` skip means keyboard navigation
  after a mouse hover won't auto-scroll if `store.input` hasn't been
  flipped back. Worth verifying that whoever consumes `store.input` flips
  it back to `"keyboard"` on the next key event — otherwise long lists
  could get stuck without keyboard-driven recentering.
- The `setTimeout(...)` inside the createEffect is preserved. The
  abort-via-stale-input pattern is correct but the underlying queue is
  still there. If the timer is 0ms it's fine; if it's longer, two rapid
  prop changes can still both be queued and only the second aborts — minor.

## Verdict

**merge-as-is**. Idempotence guard + mouse-source skip + stale-prop abort
together are the right three independent gates for the bug class. Fix is
proportional and located at the source of the loop, not at the consumer.
