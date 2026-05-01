# sst/opencode #25219 — fix: tui list jank issue

- **PR**: https://github.com/sst/opencode/pull/25219
- **Head SHA**: `bfb76d296a0081c83c3fba79b33fcd26a6c4a763`
- **Files reviewed**: `cli/cmd/tui/component/dialog-provider.tsx`, `cli/cmd/tui/component/dialog-session-list.tsx`, `cli/cmd/tui/ui/dialog-select.tsx`
- **Date**: 2026-05-01 (drip-229)

## Context

Fixes a real Solid-JS lifecycle hazard in the TUI dialog list. The
`gutter` slot on a `DialogSelectOption` was typed as
`gutter?: JSX.Element` and producers stored a *constructed* node
(`<text>✓</text>` for provider rows, `<Spinner />` for active sessions)
inside the long-lived option-data array. Solid's JSX produces real
render nodes, not React-style descriptors — once the parent `<For>`
filters or reorders, the same node instance can be detached and then
re-mounted under a different parent slot, which causes the visible
"jank" (gutter glyph briefly missing, spinner restarts, layout
recalculation flash).

## Diff (3 files, +5 -5)

`ui/dialog-select.tsx:42-45`:

```diff
-  gutter?: JSX.Element
+  gutter?: () => JSX.Element
```

`ui/dialog-select.tsx:407-422`:

```diff
-  gutter?: JSX.Element
+  gutter?: () => JSX.Element
       ...
       <Show when={!props.current && props.gutter}>
         <box flexShrink={0} marginRight={0}>
-          {props.gutter}
+          {props.gutter?.()}
         </box>
       </Show>
```

Producer call sites:

`component/dialog-provider.tsx:54`:

```diff
-          gutter: connected && onboarded() ? <text fg={theme.success}>✓</text> : undefined,
+          gutter: connected && onboarded() ? () => <text fg={theme.success}>✓</text> : undefined,
```

`component/dialog-session-list.tsx:171`:

```diff
-          gutter: isWorking ? <Spinner /> : undefined,
+          gutter: isWorking ? () => <Spinner /> : undefined,
```

## Observations

1. **Correct contract flip.** The slot is now a *factory* (`() => JSX.Element`)
   rather than a *value*. Each `<Option>` render calls `props.gutter?.()`
   at `:425`, producing a fresh node bound to the current `<box>` parent.
   This is the idiomatic Solid escape hatch when the same JSX shape needs
   to appear in N positions and the data array outlives any one of them.

2. **`Show when={!props.current && props.gutter}` is still correct.** The
   `props.gutter` in the predicate is now a function reference (truthy),
   not a node — so the gate fires whenever the producer supplied a
   factory at all. The body's `props.gutter?.()` invokes only when the
   gate has already passed; the optional chaining is defensive but not
   load-bearing. Fine.

3. **Type-checked at the source of truth.** Updating `DialogSelectOption.gutter`
   at `ui/dialog-select.tsx:42` propagates through TypeScript to both
   producer files — the diff would not compile if any other producer in
   the tree still passed a node. That's the right place to pin the
   contract.

## Nits

- No regression test. A unit test that mounts the dialog with N options,
  filters, and asserts that `gutter` factories are re-invoked (not the
  same node moved) would lock the behavior in. Without it, a future
  "small simplification" back to `gutter: JSX.Element` would silently
  bring the jank back.

- `margin?: JSX.Element` on the same interface (`:46`) has the same
  shape and presumably the same hazard if any producer ever stores a
  constructed node in long-lived option data. Worth flipping to
  `() => JSX.Element` in the same PR for symmetry.

## Verdict

`merge-after-nits` — the fix is correct and minimal at the right
boundary (factory at the slot type, factory at every producer, single
invocation site in `<Option>`). Nits are a regression test pinning
"factory invoked per render" and the symmetric `margin` flip.
