# sst/opencode PR #25345 — fix(opencode): fix infinite selection loop when hovering in TUI menus

- URL: https://github.com/sst/opencode/pull/25345
- Head SHA: `127acb24918c06a743e487950940268e6221767c`
- Author: thoughtlesslabs
- Verdict: **merge-after-nits**

## Summary

Breaks an "infinite ping-pong" reactivity loop that could occur in the TUI autocomplete and dialog-select components when the user moved the mouse quickly: the mouse-driven selection update would re-trigger the auto-scroll-to-selected effect, which would scroll, which would change which row was hovered, which would update the selection again, and so on. Fixes are: (a) in both `Autocomplete.moveTo` and `DialogSelect.moveTo`, short-circuit when `next === store.selected`, and skip auto-scroll when the input source is `mouse`; (b) in `DialogSelect`'s filter/current effect, ignore stale timeout-queued runs whose `props.current` no longer matches.

## Line-level observations

- `packages/opencode/src/cli/cmd/tui/component/prompt/autocomplete.tsx` line 484: `if (store.selected === next) return` — the early return is correct and cheap. Note that this also short-circuits the scroll math even when called programmatically with the same index, which is the intended idempotency.
- `autocomplete.tsx` line 487: `if (store.input === "mouse") return` — this is the key fix. Mouse-driven moves now never auto-scroll, breaking the loop. The semantics are: "trust where the user pointed; do not move the viewport under them." That matches user expectation for hover-driven UIs.
- `packages/opencode/src/cli/cmd/tui/ui/dialog-select.tsx` lines 146–160: the `prevInput` parameter to `on(...)` is destructured into `prevFilter` / `prevCurrent`, and the timeout body now (a) bails if `props.current` drifted while the timeout was queued (`!isDeepEqual(props.current, current)`), and (b) only re-runs the filter/current branches when *that specific input* actually changed. The `isDeepEqual` guard is what neutralizes the ping-pong specifically; the comment on lines 150–152 is helpful and should stay.
- `dialog-select.tsx` lines 175–182: same pattern as autocomplete — early-return on identical index plus mouse-source guard. Good consistency.
- One subtle consideration: `store.input === "mouse"` is a string compare; if the input source state can ever be stale (e.g. mouse-then-keyboard within the same tick) the check might suppress a legitimate keyboard-triggered scroll. Probably fine because `store.input` is updated synchronously on every input event, but worth a defensive test.

## Suggestions

1. Add a regression test (even a manual repro snippet in the PR description) that demonstrates the loop with the old code and the absence of it with the new code — this kind of bug tends to come back.
2. The double-check `if (!isDeepEqual(props.current, current)) return` reads `props.current` (current value) and compares to `current` (the value captured when the effect fired). Worth a brief code comment clarifying which side is "now" vs "then" — it's not obvious at a glance.
3. Consider extracting `if (store.selected === next) return` plus the mouse guard into a small helper used by both `moveTo` implementations to avoid drift between the two components.
