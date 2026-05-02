# charmbracelet/crush #2751 — feat: add touch tool for empty files

- URL: https://github.com/charmbracelet/crush/pull/2751
- Head SHA: `bbd245f054eaf1900861897f2c38b9414a1f3a19`
- Author: @vorticalbox
- Stats: +479 / -5 across 12 files

## Summary

Adds a new `touch` agent tool that creates an empty file (and parent
directories) at a given path. Refuses to overwrite existing files.
Wires permissions, history, file-tracking, and LSP notification the
same way `write` does, plus a UI representation in `internal/ui/chat/`.

## Specific feedback

- `internal/agent/tools/touch.go:71-94` — the **double-permission**
  pattern (request-once for "outside working dir" then again for the
  actual create) is unusual. The first prompt fires only when
  `isOutsideWorkDir`; the second always fires. A single combined prompt
  would be more user-friendly and avoid two consecutive approval
  modals for the common "create a file under `~/Projects/foo/`" case
  when the project root has been temporarily redefined. Consider the
  pattern that `write.go` already uses.
- `internal/agent/tools/touch.go:97-104` — `os.Stat` then `OpenFile(...,
  O_CREATE|O_EXCL)` is a TOCTOU pattern. The `O_EXCL` correctly closes
  the race (the second `os.IsExist` check at line 122 catches the lost
  race), so this is fine — but the early `os.Stat` is then redundant
  and only exists to give a nicer error message for the directory
  case. Worth a one-line comment so a future contributor doesn't
  "simplify" by removing the `O_EXCL` guard.
- `internal/agent/tools/touch.go:112-115` — `os.MkdirAll(dir, 0o755)`
  silently creates intermediate directories without prompting for
  permission. `write.go` does the same, so this matches existing
  behaviour, but worth flagging in the tool description so a user
  knows `touch ~/safe/a/b/c.txt` will create `~/safe/a/` and `~/safe/a/b/`.
- `internal/agent/tools/touch.go:138-142` — `files.Create(ctx,
  sessionID, absFilePath, "")` writes an empty-string history entry.
  Verify this is the convention for new files in this repo (write.go
  does the same — confirmed at the call-site convention) and that
  empty-string vs nil is not later mistaken for "deleted".
- `internal/agent/tools/touch.md:1-27` — clean and short. Could mention
  the auto-mkdir behaviour explicitly under `<features>`.
- `internal/agent/tools/touch_test.go` (+174) — good coverage; would
  appreciate one explicit test for the LSP-notify hook (mocked) since
  `notifyLSPs` is a side-effect that's easy to silently drop in a
  refactor.
- `internal/proto/permission.go:+6` and `internal/proto/tools.go:+19`
  — confirm the permission-action enum value for `touch` is "write"
  (the implementation uses `Action: "write"` at lines 78 and 122),
  not a new "create" verb. Mixing verbs across tools makes audit-log
  filtering harder.
- `internal/ui/chat/file.go:+49` and `tools.go:+25` — UI rendering
  matches the `write` tool's affordances, which is exactly right.
- `internal/ui/dialog/permissions.go:+14/-2` — small surgical change
  to add the `touch` action label. Nothing to flag.

## Verdict

`merge-after-nits` — useful tool, correct file-creation semantics
(`O_EXCL` is the right choice), permissions/history/LSP wiring matches
the existing `write` tool's pattern. The double-permission prompt for
out-of-dir paths is the only behavioural ask; the rest are doc/comment
polish.
