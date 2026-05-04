# Review: charmbracelet/crush #2590

- **Title:** feat(notification): notify using osc sequences for ssh terminal
- **Head SHA:** `b8bc59b41f4352997587bdb37a4dc3cf58fdc771`
- **Scope:** +178 / -28 across 7 files (notification subsystem)
- **Drip:** drip-339

## What changed

Adds a new OSC-9/OSC-777 escape-sequence backend so notifications can be
delivered to the user's local terminal emulator when crush is run over
SSH (where native desktop notifiers are unreachable). New file
`internal/ui/notification/osc.go` (+62), new test cases in
`notification_test.go` (+87 / -4).

## Specific observations

- `internal/ui/notification/osc.go` (+62 / -0): pure additive backend.
  Verify the OSC sequences are gated on a TTY check — emitting raw
  `\x1b]9;...\x07` to a redirected stdout will pollute logs/CI output.
- `internal/ui/notification/notification.go` (+6 / -4): the dispatcher
  is updated to fall back to OSC when the native backend is unavailable.
  Confirm the order is `native → osc → noop`; an SSH user with a working
  X-forwarded notifier should still prefer the native path.
- `internal/ui/notification/native.go` (+13 / -10): the deletions suggest
  refactoring the entry point to share an interface with the new OSC
  backend. Make sure the existing native-platform tests still cover the
  refactored signature.
- `internal/ui/notification/notification_test.go` (+87 / -4): solid bump
  in coverage. Worth a focused subtest that asserts the *exact bytes*
  emitted for an OSC-9 notification (terminator `\x07` vs `\x1b\\`)
  because some emulators (kitty, wezterm) require BEL while xterm-VT100
  honors ST. A table-driven case here would prevent regressions.
- `internal/ui/common/capabilities.go` (+1 / -1): tiny capability flag
  change — confirm any downstream consumer of `Capabilities` (e.g. status
  bar) still compiles cleanly.

## Risks

- OSC bytes leaking into non-TTY output if the gating check is missing.
- Emulator-specific terminator differences could mean some users see no
  notification and others see literal escape codes.

## Verdict

**merge-after-nits** — confirm TTY gating, add an exact-bytes assertion
in the OSC test, and document the supported terminator list.
