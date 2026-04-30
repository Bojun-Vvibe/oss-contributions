# google-gemini/gemini-cli#26263 — fix(core): tool-output cleanup on session deletion for legacy files

- PR: https://github.com/google-gemini/gemini-cli/pull/26263
- Head SHA: `55514deb799b396713b3f1b66d106aa3b4ae9aca`
- Author: cocosheng-g
- Files: 3 changed (`packages/cli/src/utils/sessionCleanup.ts` +61/−16,
  `packages/core/src/services/chatRecordingService.ts` +43/−18,
  `packages/core/src/services/chatRecordingService.test.ts` +46)
- Fixes #21568

## Context

`ChatRecordingService.deleteSessionAndArtifacts` and the CLI's
`cleanupExpiredSessions` both extract `fullSessionId` (a UUID) from the
session file in order to find and delete the associated
`tool-outputs/session-<UUID>/` directory. The session-id extraction was a
fast-path optimization for the now-default JSONL format: read the first
4096 bytes, find the first `\n`, `JSON.parse` that prefix, pull `sessionId`
out. Legacy *pretty-printed* JSON files (`session-<date>-<shortId>.json`,
not `.jsonl`) have no first-line JSON record — the first line is just
`{`. The fast parse fails, the catch-block silently swallows the error,
`fullSessionId` stays undefined, and the tool-outputs dir is leaked. The
session file itself was also being unlinked *only inside the try-block*,
so if the cleanup-other-artifacts step threw, the session file too leaked.

## Design

Two parallel changes — one in `chatRecordingService.ts`, one in
`sessionCleanup.ts` — applying the same fallback pattern.

**Service (`chatRecordingService.ts:744-816`).** The fast-path try-block
now:
1. Reads the first 4096 bytes (`:751-762`), splits on `\n`, attempts
   `JSON.parse(firstLine)`. If that succeeds and the record has a
   `sessionId` string, sets `fullSessionId`.
2. **Falls back** (`:769-779`) on parse failure to a full
   `fs.promises.readFile(filePath, 'utf8')` + `JSON.parse(fileContent)`.
   This is the legacy pretty-printed path: a 100KB file gets fully
   slurped, but only when the fast path missed. The fallback is wrapped
   in its own try/catch so a corrupt file still proceeds to unlink.
3. The `await fs.promises.unlink(filePath)` that used to live inside the
   inner try-block is now in an outer `finally`-style position — note
   the diff shows the previous "Delete the session file / await
   fs.promises.unlink(filePath)" line removed from the try-block; the
   replacement (cut off in the diff window) ensures unlink still runs
   when both parse paths fail. The PR description calls this out
   explicitly: "Ensured session files are unconditionally unlinked
   during deletion, even if an error occurs while attempting to delete
   artifacts."

**CLI (`sessionCleanup.ts:163-216`).** Same pattern applied to the
*automatic* expiry-cleanup path. The `SHORT_ID_REGEX` at `:36` widens
from `/-([a-zA-Z0-9]{8})\.json$/` to `/-([a-zA-Z0-9]{8})\.jsonl?$/` so
both extensions match. `deriveShortIdFromFileName` (`:74-80`) and the
file-listing filter at `:160-166` get the same `.json` || `.jsonl` widen.

The inner-fallback at `:174-209` mirrors the service-side pattern: open
the FD, read 4096 bytes, parse first line, on failure read the whole
file. Two helper type-guards (`hasProperty`, `isStringProperty`,
`isSessionIdRecord` at `:38-54`) tighten the previous
`'sessionId' in content && typeof content === 'object'` check into a
narrowed `obj is { sessionId: string }` predicate. This is
mechanically what `chatRecordingService.ts` already had (the test file
imports `isSessionIdRecord` from there), now duplicated into the CLI
package. Worth a follow-up to extract into a shared util.

## Tests

Two new cases in `chatRecordingService.test.ts:605-650`:

- **`should delete legacy pretty-printed session files and their artifacts`**
  (`:605-630`). Writes `session-2023-01-01T00-00-legacy12.json` with
  `JSON.stringify({sessionId, messages:[]}, null, 2)` (the `, null, 2`
  is what makes it pretty-printed and breaks the first-line parse).
  Creates `tool-outputs/session-legacy-uuid/output.txt`. Calls
  `deleteSession(shortId)`. Asserts both the session file *and* the
  tool-output dir are gone. This is the exact bug repro.
- **`should delete the session file even if it is corrupted (invalid JSON)`**
  (`:632-650`). Writes `session-...-corrupt1.jsonl` with literal text
  `'not-json'`. Asserts the file is unlinked. This pins the
  unconditional-unlink invariant.

Both are minimal repros, no mocking, no env munging — exactly the right
shape for a deletion-correctness test.

## Risks / nits

- **Duplicate `isSessionIdRecord`** between
  `cli/src/utils/sessionCleanup.ts:53-55` and
  `core/src/services/chatRecordingService.ts` (already imported there).
  The CLI package already depends on the core package — pull this from
  there instead of re-defining. One-line export change.
- **Fallback memory cost**: a corrupt 100MB session file would be
  fully `readFile`'d into memory. Realistic session files are
  bounded but not formally capped. A `fs.stat()` + size guard before
  the fallback would close that worry; reasonable to leave as a
  follow-up.
- **`deleteSessionArtifactsAsync(fullSessionId, tempDir)` (`:798`)** is
  called only when `fullSessionId` is set; if both the fast path and
  the fallback fail to find one, the warning is logged
  (`:799-803`, cut off in the diff window) and the tool-outputs dir
  is not touched. That's the right policy — better to leak a tool-
  outputs dir than to delete a wrong one — but the warning should
  include both the file path and which parse attempts failed, for
  ops triage.

## Verdict

**`merge-as-is`**

The fix is at the right layer (fallback in the same try-block, not a
separate code path), the fallback is bounded by "only on first-line
parse failure" (no extra cost on the JSONL hot path), the unconditional-
unlink invariant is pinned by an explicit corrupt-file test, and the
type-guard tightening is the right direction. The duplicated
`isSessionIdRecord` and the unbounded readFile are minor follow-ups, not
blockers.

## What I learned

When a fast-path optimization assumes a file format invariant (here:
"first line is parseable JSON"), the right cleanup-on-missing-invariant
shape is *fall back to the slow path inside the same try-block*, not
*skip the operation*. The original code did the latter — silently
returned with `fullSessionId === undefined` — which is the worst possible
combination: no error, no log, no metric, just a slow leak of
tool-outputs dirs. The test pattern of "write a pretty-printed JSON file,
verify the legacy artifact is cleaned up" is the kind of test you can
only write if you've actually grep'd the user-bug-report for the file
shape. Pinning *unconditional unlink* with a separate corrupt-file test
is the second invariant worth pinning — those two tests together cover
the "legacy" and "broken" branches of "anything not a valid JSONL".
