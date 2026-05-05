# sst/opencode #25902 — fix(tui): stop plugin selection mouse jumps

- URL: https://github.com/sst/opencode/pull/25902
- Head SHA: `92047247c45dd9bcd89bf50095316d0f8037195b` (`9204724`)
- Author: @3351163616 (XiaoYang)
- Diff: +0 / −5 (1 file: `packages/opencode/src/cli/cmd/tui/feature-plugins/system/plugins.tsx`)
- Verdict: **merge-as-is**

## What this changes

Five-line deletion that breaks a feedback loop between the plugins
dialog and `DialogSelect`. Pre-patch the `View(props)` component
maintained its own `cur` signal at line 158
(`const [cur, setCur] = createSignal<string | undefined>()`), passed
it to `DialogSelect` via `current={cur()}`, and updated it from three
sites: the `onMove` callback (every hover/keyboard cursor change), the
keybind `onTrigger` (toggle key), and `onSelect` (mouse click /
Enter). Post-patch the local signal and all three setters are gone;
`DialogSelect` is left to manage its own transient
hover/keyboard-selection state internally.

The bug this fixes: every mouse hover fired `onMove` → `setCur(value)`
→ re-render → `current={cur()}` re-passed to `DialogSelect` →
`DialogSelect` interprets a new `current` prop as "the parent has
authoritatively moved the selection to here" and re-centers the list
on the cursor's row. Because the user's pointer is still hovering
inside the list while the re-center happens, the row under the cursor
shifts, the next mouse-move event hits a different row, `onMove`
fires again with the new value, and the list jumps again. Classic
controlled-vs-uncontrolled feedback oscillation.

## Why removing both setters in `onTrigger` and `onSelect` is correct

The keybind `onTrigger.setCur(item.value)` and `onSelect.setCur(item.value)`
deletions look at first glance like they could regress
keyboard-toggle / click-toggle persistence — but `flip(item.value)` is
the actual mutation (toggles the plugin's enabled state in the
underlying store), and `DialogSelect` already tracks which row the
user just acted on internally for purposes of its own redraw. The
external `cur` signal was only ever feeding `current=` back into
`DialogSelect`, and once `current=` is gone there's nothing to keep in
sync.

## Concerns

None blocking. The author confirms manual verification of mouse
hover, click, scroll, and mixed keyboard/mouse selection in the
Plugins dialog with before/after binaries, plus a clean
`bun typecheck`. The fix is structurally a strict-uncontrolling of a
component that didn't need to be controlled — minimal viable change.

One forward-looking observation worth surfacing in the merge thread:
if any *other* TUI dialog in the same `feature-*/system/` tree uses
the same `DialogSelect` + local `cur` signal pattern, it has the same
bug. A grep for `current={cur()}` near `onMove={` across the TUI
package would catch siblings cheaply (this PR is correctly scoped to
the reported plugin-manager case from #25903 and shouldn't expand,
but the pattern is worth noting for follow-up). Canonical
narrow-fix `merge-as-is`.
