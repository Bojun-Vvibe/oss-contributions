# google-gemini/gemini-cli#26184 — `fix(core): fail session deletion when file is missing`

- URL: https://github.com/google-gemini/gemini-cli/pull/26184
- Head SHA: `fde348abab74`
- Author: haosenwang1018
- Size: +9 / -2 across 2 files
- Fixes: #26113

## Summary

`ChatRecordingService.deleteSession()` previously returned
successfully when the chats directory existed but no matching
`.jsonl` file was found. The interactive `/resume` browser
interpreted that as "deletion succeeded" and removed the row from
its in-memory list, even though nothing was deleted on disk —
on the next `/resume` open, the session reappeared, looking like
a deletion bug. This PR makes the missing-file case fail loudly so
callers can surface a real error.

## Specific references

`packages/core/src/services/chatRecordingService.ts:695-700`
(post-patch):

```ts
if (matchingFiles.length === 0) {
  throw new Error(`No session file found for ${sessionIdOrBasename}`);
}

for (const file of matchingFiles) {
  await this.deleteSessionAndArtifacts(chatsDir, file, tempDir);
}
```

The throw is positioned *after* `chatsDir` existence is
established (the diff is inside the `try` block that already
short-circuits if the chats directory itself is missing), so this
specifically targets the "directory exists, ID didn't match
anything" case — exactly the silent-failure pattern from #26113.

`packages/core/src/services/chatRecordingService.test.ts:728-737`
(post-patch):

```ts
it('should throw if session file does not exist', async () => {
  const chatsDir = path.join(testTempDir, 'chats');
  fs.mkdirSync(chatsDir, { recursive: true });

  await expect(
    chatRecordingService.deleteSession('non-existent'),
  ).rejects.toThrow('No session file found for non-existent');
});
```

The test correctly creates the chats dir so the assertion exercises
the new branch and not the pre-existing "no chats dir" path.

## Design analysis

This is a textbook "stop fail-open silently, surface to caller"
fix. The contract change is `delete(non-existent)` resolves →
`delete(non-existent)` rejects, which is the right shape for an
imperative "delete X" verb (unlike DELETE in HTTP, which is
idempotent, the in-memory UI here treats the call as a fire-and-
forget commit; idempotent-success was actively misleading).

Two things to verify:

1. **All callers handle the new rejection.** The `/resume`
   browser is the obvious one and the PR description claims it
   benefits from the change. Any other call site (e.g. cleanup
   on session-rotation, telemetry) needs a `try/catch` or an
   explicit allowlist of "missing is fine here." A one-line
   `rg "deleteSession\(" packages/` confirmation in the PR body
   would close this.
2. **Concurrent deletions** of the same session ID by two TUI
   panes will now fail the loser instead of silently succeeding.
   Probably desired (the user can refresh), but worth a sentence.

## Risks

- Behavior-changing for any caller that relied on the old
  fail-open semantics. Mitigated if the callers are all internal
  to this repo and pass through the `/resume` UI.
- Error message includes `sessionIdOrBasename`. If that argument
  ever contains user-controlled content that flows to logs,
  it's sanitized by being a session ID. No injection concern.

## Verdict

`merge-after-nits`

## Suggested nits

- Paste a `rg "deleteSession\("` listing in the PR body to prove
  no caller silently breaks on the rejection.
- Consider adding a second test pinning the existing "no chats
  directory at all" path still resolves cleanly (i.e. doesn't
  throw the new error), so the two cases stay distinct.
