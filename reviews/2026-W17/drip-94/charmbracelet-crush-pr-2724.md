# charmbracelet/crush #2724 — fix(ui): restore 'update available' notification

- Author: meowgorithm (Christian Rocha)
- Head SHA: `307ceed0804fe3e6e53f294e6250160efdbb8dff`
- Single file: `internal/ui/model/ui.go` (+12 / −0)

## Specifics

- New case at `ui.go:856-867` adds handling for `app.UpdateAvailableMsg` to the `Update` message switch. Two-branch render: production builds show `"Crush update available: vX → vY."`, dev builds show `"This is a development version of Crush. The latest version is vY."`. Both routed through `m.status.SetInfoMsg(util.InfoMsg{Type: util.InfoTypeUpdate, ...})` with a 10s TTL and the standard `clearInfoMsgCmd(ttl)` follow-up.
- The fact that this is a "restore" implies a previous regression where the case fell through and the message was silently dropped — the new branch slots in cleanly above `case util.ClearStatusMsg:` (`ui.go:868`), preserving switch ordering.
- Uses `util.InfoTypeUpdate` (presumably distinct from `InfoTypeError` / `InfoTypeInfo`) so the status component can style update notices differently. Good separation.
- `IsDevelopment` branching is a nice UX touch: dev builds shouldn't nag users to "update" when they can't, instead noting the upstream version for context.

## Concerns

- 10-second TTL is hardcoded inline rather than reusing `DefaultStatusTTL` (used in the immediately-preceding case at `ui.go:854`). Either match the default or extract `UpdateNotificationTTL` so the magic number isn't load-bearing.
- No test. Given this is literally a switch-case wired to a typed message, a `tea`-based table test asserting `Update(UpdateAvailableMsg{...})` produces the expected `InfoMsg` would catch any future case-deletion regression that caused the original "restore" need in the first place.
- The string `"Crush update available: v%s → v%s."` and the dev variant are user-visible English with no i18n hook; consistent with the rest of the file based on the surrounding context, so non-blocking.

## Verdict

`merge-as-is` — small, correct restoration of a missing case. The TTL nit and missing test are minor; project lead self-merge is appropriate.

