---
pr_number: 2646
repo: charmbracelet/crush
head_sha: cf604cf13722446e71af26e3fc9d379b7f52c8ff
verdict: merge-after-nits
date: 2026-04-25
---

# charmbracelet/crush#2646 — CamelHumps editing for `ctrl+left/right/backspace/delete`

**What changed.** +584 / −11 across `internal/ui/model/keys.go`, `internal/ui/model/layout_test.go`, and presumably `internal/ui/model/keys_handlers.go` (truncated in the diff window). Four new `key.Binding`s (lines 140–151 of `keys.go`): `MoveWordBackward = ctrl+left`, `MoveWordForward = ctrl+right`, `DeleteWordBackward = ctrl+backspace`, `DeleteWordForward = ctrl+delete`. The new bindings have *no* `WithHelp` text — only `WithKeys` — meaning they don't surface in the help/keymap overlay. The `KeyMap.Editor` struct gains four corresponding fields (lines 7–17). Tests at `layout_test.go:117–250` cover three navigation patterns: word-by-word (`one two three` → columns 8, 4 backward), CamelCase (`parseHTTPResponse` splits at `parse|HTTP|Response`), and snake_case + CamelCase mix (`foo_barBaz` splits at `foo|_bar|Baz`).

**Why it matters.** Body says: existing `alt+left/right` does whole-word, `ctrl` shortcuts now do subword. For working in code-heavy prompts (paths, identifiers, snake_case env names), CamelHumps navigation matches what every modern IDE provides. The new shortcuts route through the UI editor layer (so `@` completion popups and draft state stay in sync after a subword delete), not directly to the underlying textarea — that's the right place for it.

**Concerns.**
1. **`MoveWordBackward`/`MoveWordForward`/`DeleteWordBackward`/`DeleteWordForward` have no `WithHelp(...)`** (keys.go:140–151). Every other binding in the file follows `key.WithKeys(...) + key.WithHelp(...)`. Consequence: `?` overlay won't list them. For a feature whose main user benefit is *discoverability* of keystrokes, this is a regression vs. the alt-prefixed pair. Add `key.WithHelp("ctrl+←/→", "move by subword")` and `key.WithHelp("ctrl+⌫/⌦", "delete subword")`.
2. **CamelHumps boundary semantics for digits and punctuation are not tested.** Tests cover whitespace, snake, camel, ALLCAPS-acronym. Not covered: `parse2HTTPResponse` (digit boundary), `parse-http-response` (kebab), `parseHTTP-Response` (mixed), `IPAddress` (acronym followed by uppercase head), trailing/leading underscores. The implementation file is truncated in the diff but at +584 lines clearly includes the segmentation algorithm — adding 2-3 cases for digits and kebab is cheap insurance.
3. **`ctrl+backspace` on macOS terminal emulators** is notoriously inconsistent. Terminal.app sends `\b` (same as `backspace`), kitty/iTerm2 send the bracketed sequence. Bubbletea normalizes most of these, but if the binding only fires on the bracketed form, mac users on Terminal.app will see no behavior change. A help-line + a release-notes mention saying "may require iTerm2/kitty/wezterm" prevents bug reports.
4. **`newTestUI()` now constructs the full UI surface** (dialog overlay, completions component, attachments component with keymap) — previously a minimal stub (lines 32–69 vs. the prior 4-field literal). That's the right move for tests that exercise the editor through the UI layer. But it makes `newTestUI` non-trivial and shared across many tests; confirm none of the existing tests in the file rely on the absent overlays being `nil` (which would now be `dialog.NewOverlay()` — non-nil empty).
5. **`Editor.MoveWordBackward` field name** is the same word the textarea component uses for whole-word; subword vs. word naming asymmetry could confuse future maintainers. `Editor.MoveSubwordBackward` reads better given `alt+left` already owns word semantics.

Solid feature, well-tested for the common cases. Land after the help-text fix.
