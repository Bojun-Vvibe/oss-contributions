# google-gemini/gemini-cli #26184 — fail session deletion when file is missing

- **PR:** https://github.com/google-gemini/gemini-cli/pull/26184
- **Title:** `fix(core): fail session deletion when file is missing`
- **Author:** haosenwang1018
- **Head SHA:** fde348abab749d54438ccd47c9363f6eb997d01e
- **Files changed:** 2 (`packages/core/src/services/chatRecordingService.ts`,
  `packages/core/src/services/chatRecordingService.test.ts`), +9 / −2
- **Verdict:** `merge-as-is`

## What it does

Fixes #26113. `ChatRecordingService.deleteSession()` previously resolved
successfully when no session file matched the requested id — the
interactive `/resume` browser interpreted that as "deletion succeeded"
and removed the row from UI state, while the underlying `.jsonl` stayed
on disk. Result: phantom-deleted sessions reappeared on the next launch.

The fix in `chatRecordingService.ts:698-700`:

```ts
if (matchingFiles.length === 0) {
  throw new Error(`No session file found for ${sessionIdOrBasename}`);
}
for (const file of matchingFiles) {
  await this.deleteSessionAndArtifacts(chatsDir, file, tempDir);
}
```

The test at `chatRecordingService.test.ts:731-737` is updated to assert
the **opposite** of its previous assertion: now expects a rejection with
`'No session file found for non-existent'`.

## What's good

- Correct root-cause fix at the service boundary, not at the UI layer.
  The `/resume` browser is now allowed to be naive ("if delete throws,
  show an error and keep the row") because the service is honest about
  what it actually did.
- The error message includes the supplied `sessionIdOrBasename`, which
  is exactly what a user troubleshooting a "session won't delete" bug
  needs to see in logs.
- The test setup explicitly creates `chatsDir` first
  (`fs.mkdirSync(chatsDir, { recursive: true })`) so the test exercises
  the empty-match path inside an existing chats directory — that's the
  case that the bug actually hit, not the "chats dir doesn't exist"
  case which is a different code path higher up.
- Two-line code change, no API surface change, no migration. Callers
  already had to handle exceptions from `deleteSessionAndArtifacts`
  (which can throw on `unlink` failures), so adding one more throw
  reason doesn't change the contract callers must already implement.

## Nits / risks

- The new error is a generic `Error`. Consumers that want to
  distinguish "no such session" from "fs error" will have to
  string-match the message, which is fragile. A small typed error
  (`class SessionNotFoundError extends Error {}`) would be friendlier
  to the `/resume` UI code and to any future SDK consumers. Not
  blocking, but worth a follow-up.
- The PR doesn't update `/resume` browser code to display the new
  error. If the existing `/resume` handler currently swallows
  exceptions silently, the user will see "row removed, then row
  reappears on reload" — a worse UX than before. Worth confirming
  there's at least a `toast.error` or `console.error` somewhere in
  the call chain. Quick `grep deleteSession` would tell us; the diff
  doesn't include that file so I can't verify from this PR alone.
- No test for the case where `matchingFiles.length > 0` and the
  underlying `deleteSessionAndArtifacts` throws partway through (e.g.
  one file deletes, second one EPERM). The new throw doesn't
  regress that behavior, but a test that one partial-failure scenario
  would be valuable defense in depth.

## Verdict rationale

`merge-as-is`: minimal, correct fix at the right layer with a test that
actually verifies the changed behavior. The nits are all in the
"could-be-nicer follow-ups" category — none warrant blocking a 9-line
fix to a real user-visible bug.
