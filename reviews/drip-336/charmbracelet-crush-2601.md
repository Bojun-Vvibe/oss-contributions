# charmbracelet/crush #2601 — fix: refresh TUI when session is updated by an external process

- **Head SHA reviewed:** `451a99a7f7325ef9978b19ee2049388499ab60db`
- **Size:** +169 / -0 across 3 files
- **Verdict:** merge-after-nits

## Summary

The crush TUI uses an in-process pubsub broker for session updates, so
when a second process mutates the same session (e.g.
`crush run --continue` in another terminal), the open TUI never sees
the new messages until the user manually switches sessions. This PR
adds a 1 s polling tick that compares the cached `session.UpdatedAt`
to a fresh DB read; if it advanced, the TUI emits an
`externalSessionChangedMsg` to drive a full reload.

## What I checked

- `internal/ui/model/session.go:251-292` (added) — three small pieces:
  `externalUpdateInterval = time.Second` constant,
  `checkExternalUpdateMsg`/`externalSessionChangedMsg` message types,
  `scheduleExternalUpdateCheck()` (a `tea.Tick` wrapper), and
  `(m *UI) checkExternalSessionUpdate()` which calls
  `m.com.Workspace.GetSession(ctx, sessionID)` and compares
  `sess.UpdatedAt > knownUpdatedAt`.
- `internal/ui/model/ui.go` (+15 lines) — wires the tick into the
  bubbletea Update loop; not shown in the diff snippet I read, but
  the test fixture (`newTestUIWithSession`) confirms the wiring.
- `internal/ui/model/session_test.go:1-110` — three good unit tests:
  `DetectsChange` (cached=100, fresh=200 → emits message),
  `NoChangeWhenTimestampUnchanged` (returns nil),
  `NoChangeWhenTimestampGoesBack` (cached=100, older=50 → returns
  nil). Solid coverage of the comparison logic.

## Concerns / nits

1. **1 s polling cost.** Every active TUI now issues one
   `Workspace.GetSession` call per second per open session. For a
   single-user workstation this is invisible; if a `Workspace` is
   backed by anything more expensive than a SQLite read (network DB,
   future remote workspace), the cost adds up. Worth either (a)
   exposing the interval as a config value or (b) backing off when
   the TUI is idle / unfocused.
2. **Polling vs. fs notify.** The session store ultimately writes to
   a known path; if the workspace already has a deterministic file
   layout, an `fsnotify` watcher would be both cheaper and more
   responsive than a 1 s tick. Not blocking, but worth a follow-up
   discussion — comment in code that polling was the deliberate
   simple choice would help.
3. **Cancel-on-leave.** `scheduleExternalUpdateCheck` returns a
   `tea.Cmd` that fires *one* tick. Make sure the Update loop only
   re-schedules while the TUI is still owning that session — if the
   user switches sessions, the in-flight tick still runs once and
   compares the *old* `m.session` against the now-stale
   `knownUpdatedAt`. The captured closure (`sessionID := m.session.ID`,
   `knownUpdatedAt := m.session.UpdatedAt`) at line 277 is a snapshot,
   so it's safe — but a brief comment would help future maintainers
   not "fix" it by reading from `m.session` inside the closure.
4. **Error swallowed at debug.** `slog.Debug("External update check: failed to get session", …)`
   is fine for steady-state, but a transient workspace error during
   shutdown can spam the log at debug level. Consider rate-limiting
   the log line or escalating to warn after N consecutive failures.
5. The test uses `&sessionWorkspace{Workspace: workspace.Workspace{}}`
   embedding pattern; `Workspace` is presumably an interface or empty
   struct. If it's an interface, the embedding may shadow non-mocked
   methods with nil dispatch; if so, prefer a named-method-only mock.

## Risk

Low. Polling adds steady CPU/IO, but the comparison is monotonic and
the closure correctly snapshots the cached values. Worst case is
extra DB load on a misconfigured remote workspace.

## Recommendation

Merge after a one-line code comment on the deliberate snapshot
behavior at `session.go:277-278` and confirmation that the parent
Update loop re-schedules only for the active session. Polling
interval as config can be a follow-up.
