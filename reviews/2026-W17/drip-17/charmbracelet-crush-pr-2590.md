# charmbracelet/crush PR #2590 — feat(notification): notify using OSC sequences for ssh terminal

- **Author:** nghiant03
- **Head SHA:** 53dc9151bdf8ee9c6f87e14b343c22468b7bfe02
- **Files:** 7 under `internal/ui/notification/`,
  `internal/ui/common/capabilities.go`, `internal/ui/model/ui.go`
  (+178 / −28)
- **Verdict:** `merge-after-nits`

## Context

Desktop notifications today use `gen2brain/beeep` (native OS toast).
Over SSH that does nothing useful — the toast appears on the *remote*
host's desktop, not the user's. This PR adds an `OSCBackend` that
emits OSC 99 + OSC 777 escape sequences over the terminal stream so a
modern terminal emulator on the *local* machine raises the
notification. SSH detection is via `SSH_TTY`.

## What the diff does

- `internal/ui/notification/notification.go` lines 12–17: the
  `Backend` interface changes from `Send(n) error` to
  `Send(n) tea.Cmd`. Each backend now controls its own execution
  model — native runs in a goroutine inside the returned `tea.Cmd`,
  OSC returns a `tea.Raw` command that bubbletea writes inline.
- `internal/ui/notification/osc.go` (new, 62 lines): builds a single
  string with three writes — OSC 99 title, OSC 99 body, optional
  OSC 99 base64-encoded icon, terminating OSC 99 sentinel
  (`d=1`), then OSC 777 (`URxvtExt("notify", title, message)`).
  Returns `tea.Raw(sb.String())`. IDs are `crush-<atomic-counter>`.
- `internal/ui/model/ui.go` lines 478–485: when capabilities query
  comes back, if `SSH_TTY` is set use `NewOSCBackend`, otherwise
  `NewNativeBackend`.
- `internal/ui/common/capabilities.go` lines 73–77:
  `RequestModeFocusEvent` is moved out of the "smart terminal"
  branch and sent unconditionally. This is the trigger that makes
  `tea.ModeReportMsg` reliably arrive so the SSH branch above can
  fire.

## Review notes

- The interface change `error → tea.Cmd` is the right architectural
  call: native blocks on a syscall, OSC is a write. Trying to fit
  both behind `error` would force the caller to spawn a goroutine
  for every backend, which is what the old code did at
  `model/ui.go:404-411` (now removed). Cleaner.
- `osc.go:46` — the OSC 99 sentinel `d=1` is the "complete"
  marker. The order is title → body → icon → sentinel → OSC 777.
  Some terminals (foot, wezterm) ignore unknown OSC 99 attributes
  but a few older emulators may render the title twice if the body
  arrives after a "complete" sentinel from a prior notification.
  The atomic `notifySeq` ID handles this correctly because each call
  uses a fresh `i=` value.
- `osc.go:21` — `notifySeq` is package-level. Across multiple
  bubbletea programs in the same process this is fine; if crush
  ever embeds itself, the global counter is an information leak
  (number of notifications since process start). Trivial, not
  blocking.
- `model/ui.go:478` — the `SSH_TTY` check is one-shot at
  `tea.ModeReportMsg` time. If the user starts crush over SSH then
  re-attaches locally (e.g. tmux), the backend won't switch. Worth
  a comment, since the alternative (re-detecting on every send)
  isn't obviously better.
- The capabilities.go diff is a real behaviour change for every
  user, not just SSH. Sending `RequestModeFocusEvent`
  unconditionally means dumb terminals will receive an escape they
  don't recognise. Most will ignore it; a few will print garbage
  on first paint. This deserves a one-line justification in the
  commit message.
- Tests at `notification_test.go:90-127` exercise OSC 99 + OSC 777
  + icon base64 — solid coverage for the happy path. Missing: a
  test that confirms `tea.RawMsg` is what's wrapped (the helper
  asserts it but only as a positive case).

## What I learned

The OSC 99 + OSC 777 dual-emit pattern is a nice example of
deliberately-redundant transport: cheap to send both (a few hundred
bytes), and the union of supporting terminals covers most modern
emulators. The capabilities-query side-effect is the load-bearing
part — without making `tea.ModeReportMsg` reliably fire, the SSH
branch never runs.
