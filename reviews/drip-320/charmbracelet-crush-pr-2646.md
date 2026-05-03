# charmbracelet/crush PR #2646 — feat(ui): add CamelHumps editing for ctrl word shortcuts

- Link: https://github.com/charmbracelet/crush/pull/2646
- SHA: `cf604cf13722446e71af26e3fc9d379b7f52c8ff`
- Author: enrell
- Stats: +584 / −11, 5 files

## Summary

Adds CamelHumps-aware editing on `ctrl+left/right/backspace/delete` while preserving the existing whole-word semantics on `alt+left/right`. The `KeyMap` is extended with four new bindings, and the textarea editor layer routes the new shortcuts through a custom delete/move pipeline so prompt draft state and `@` mention completions stay consistent after subword deletions.

## Specific references

- `internal/ui/model/keys.go` L7–L18: four new fields added to `KeyMap.Editor` (`MoveWordBackward`, `MoveWordForward`, `DeleteWordBackward`, `DeleteWordForward`). Naming is consistent with the existing struct.
- `internal/ui/model/keys.go` L137–L150: new bindings only set `WithKeys`, no `WithHelp`. The neighbouring bindings (e.g. `Commands`, `AttachmentDeleteMode`) all set `WithHelp` so the help overlay renders them. These four will be invisible in `?` help — likely intentional (they are conventional editor shortcuts), but worth a one-line decision in the PR description.
- `internal/ui/model/textarea_delete.go` and `internal/ui/model/textarea_keys.go` (new files): the routing of the new keys lives here. The diff shows the test harness `newTestUI()` is being expanded with `dialog`, `completions`, and `attachments` to make the new keymap exercisable in `layout_test.go`. Good — without that, the integration would be untested.
- `internal/ui/model/layout_test.go` L32–L75: `newTestUI()` now wires `dialog.NewOverlay()`, `completions.New(...)`, and `attachments.New(...)`. This is a meaningful expansion of the test harness; future contributors should be able to reuse it. Confirm the helper is intended to be the canonical test UI.
- `internal/ui/model/ui.go`: not inspected in detail, but the keymap dispatch must route to the new `textarea_delete.go` helpers; verify there is no double-dispatch with the existing `alt+...` bindings on terminals where ctrl and alt sequences overlap (e.g. some macOS Terminal.app configurations).

## Verdict

verdict: merge-after-nits

## Reasoning

User-visible quality-of-life feature with a real test-harness investment. Two nits worth addressing before merge: (1) decide whether the four new bindings should appear in the `?` help overlay (`WithHelp` calls), and (2) add a comment in `keys.go` documenting that `ctrl+*` is intentionally CamelHumps-aware while `alt+*` stays whole-word, so the distinction survives future refactors.
