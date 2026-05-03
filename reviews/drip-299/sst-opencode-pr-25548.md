# sst/opencode PR #25548 — fix: Add Windows support for Zed database path resolution

- URL: https://github.com/anomalyco/opencode/pull/25548
- Head SHA: `ef88ada56b467021151fff0739ed8f1593587602`
- Author: erroris3 (Nitipoom Aumpitak)

## Summary

Adds the Windows `%LOCALAPPDATA%\Zed\db\0-stable\db.sqlite` candidate to
`resolveZedDbPath()` in `packages/opencode/src/cli/cmd/tui/context/editor-zed.ts:185`,
so opencode's Zed editor context bridge can find the local DB on Windows.

## Review

The single-line addition slots into the existing `candidates` list at the
correct precedence (after the `OPENCODE_ZED_DB` env override, before the
macOS/Linux defaults), uses `os.homedir()` + `path.join` like the siblings,
and `.filter(Boolean)` already strips nulls — no further changes needed.

Nits worth flagging on the PR:

1. Zed on Windows actually stores the DB at
   `%LOCALAPPDATA%\Zed\db\0-stable\db.sqlite`, which under `os.homedir()` on
   Windows resolves correctly via `AppData\Local\Zed\...`. So the path is
   right for the default install, but if the user has redirected
   `LOCALAPPDATA` (corp policy / junctioned profile), `path.join(homedir(),
   "AppData", "Local", ...)` will silently miss it. Consider preferring
   `process.env.LOCALAPPDATA` when set, then falling back to the joined path.
2. The dev/preview channel uses a different stable label (`0-preview`); not
   in scope for this PR but the Linux/macOS candidates have the same gap, so
   either leave a TODO or punt to a follow-up.

## Verdict

`merge-after-nits` — accept the addition; the `LOCALAPPDATA` env preference
would make the Windows path as robust as the macOS/Linux ones, but it's not
a blocker.
