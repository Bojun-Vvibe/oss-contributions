# charmbracelet/crush #2742 — chore(ui): change wording: rewrote input to rewrote output

- Link: https://github.com/charmbracelet/crush/pull/2742
- Head SHA: `58b2923180c3e35f3e395b805d2c2320894378bf`
- State: MERGED 2026-04-30, +11/-6 across 1 file

## Summary
Tiny but well-executed cleanup. The hook-result line in the chat UI
displayed "Rewrote Input" when a hook rewrote a tool result, which is
incorrect — by the time hooks run, that string represents the tool's
**output** the hook is modifying. The PR (a) renames the displayed text
to "Rewrote Output" and (b) extracts the three repeated string literals
("OK", "Denied", "Rewrote Output") into named constants at the top of
`hookDetail()` so the same string isn't re-typed in three switch arms.

## Specific references
- `internal/ui/chat/tools.go:800-803` — declares `okMessage`, `denialMessage`,
  `rewroteMessage` as a `const ( … )` block at the top of `hookDetail`.
- `internal/ui/chat/tools.go:807-820` — switch arms now reference the
  constants instead of inline literals; the `"allow"` and `default` arms
  use `rewroteMessage` (was "Rewrote Input").

## Notes
- Pure UI string. No behavior change beyond the displayed text.
- Could have lived as a top-level package constant rather than nested inside
  the function, but the local scope is fine for three single-use messages
  and avoids polluting the package namespace.
- Already merged.

## Verdict
`merge-as-is`
