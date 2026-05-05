# google-gemini/gemini-cli #26514 — feat: export session to file and import via flag

- URL: https://github.com/google-gemini/gemini-cli/pull/26514
- Head SHA: `1abeb145d062b2a38610d066ec725b51d686f0e5` (`1abeb14`)
- Author: @cocosheng-g (Coco Sheng)
- Diff: +298 / −1 across 6 files
- Verdict: **request-changes**

## What this changes

Closes #23663. Adds a paired `/export-session <path>` slash command
plus `--session-file <path>` CLI flag for the round-trip. Concretely:

- `packages/cli/src/ui/commands/exportSessionCommand.ts` (+78, new) —
  the export side, resolving the active session via `SessionSelector`
  and writing the raw JSON `ConversationRecord` to the requested
  path.
- `packages/cli/src/services/BuiltinCommandLoader.ts` (+2) — wires the
  new command into the loader's command list (alphabetically between
  `docs` and `directory`, per the diff).
- `packages/cli/src/config/config.ts` (+6) — adds the
  `--session-file` yargs string flag and the `sessionFile?: string`
  field on the `CliArgs` interface at line 100.
- `packages/cli/src/gemini.tsx` (+54 / −1) — extends
  `resolveSessionId` with an optional `sessionFileArg` parameter,
  threads `argv.sessionFile` from `main()` at line 395, and
  implements the import branch at lines 215-260.
- `packages/cli/src/gemini.test.tsx` (+46) — one new test
  `'should import from session file when sessionFile is provided'`
  asserting (a) a fresh `sessionId` is allocated, (b) the conversation
  record's `sessionId` field is rewritten to match, (c) the new
  `filePath` lives under the temp-chats dir keyed by the new id's
  prefix.
- `packages/cli/src/ui/commands/exportSessionCommand.test.ts` (+112,
  new) — covers the export side.

## Why the import shape is correct (the load-bearing design call)

The import branch at `gemini.tsx:215-258` deliberately treats the
input file as **read-only seed data**, never the live session storage.
On import: load the file via `loadConversationRecord()`, allocate a
brand-new local `sessionId` via `createSessionId()`, recompute
`projectHash` against the *current* project root via
`getProjectHash(storage.getProjectRoot())`, set fresh `startTime`/
`lastUpdated` to now, then materialize the conversation under
`storage.getProjectTempDir()/chats/session-{now}-{id8}.jsonl` and
return that as the live `filePath`. The original file passed via
`--session-file` is untouched — subsequent edits go to the new local
JSONL, exactly per the PR's stated contract.

This is the right design because it lets a session be shared across
machines/projects (export from project A → drop the JSON in a
ticket → import into project B) without one user's continuation
silently mutating another user's source-of-truth file, and without
the project-hash mismatch causing weird "this session belongs to a
different project" warnings. It also means an imported session that
lives in a wrong-shape JSON file fails loud (caught by the
`try`/`catch` at line 218) rather than corrupting a real on-disk
session.

## Why this is `request-changes`

Three blocking issues, in order of severity.

**(1) The export-side file write has no path-traversal or
write-permission guard.** The new export command (per the PR body)
"writes the raw JSON `ConversationRecord` to the requested path." If
the user types `/export-session ../../../etc/gemini-leak.json` or
`/export-session /tmp/leak.json` from a session running with elevated
privileges (the gemini-cli is regularly invoked from CI / agent
runners), the conversation record — which can contain pasted secrets,
file contents, tool outputs — gets written wherever the user-or-model
specifies. The export command should at minimum:

- Reject absolute paths outside the user's home / CWD without an
  explicit `--force` confirmation.
- Refuse to overwrite an existing file without `--overwrite`.
- Set `0o600` mode on the written file (conversations contain
  secrets).

The diff for `exportSessionCommand.ts` wasn't fully visible in my
fetch slice, but the pattern is verifiable from the test at
`exportSessionCommand.test.ts` and the PR body's "writes the raw
JSON ... to the requested path" wording — neither suggests these
guards exist.

**(2) The new test has timing-fragile assertions and a noisy
process.exit handler.** At `gemini.test.tsx:1093-1110` the test wraps
the call in `try/catch (e instanceof MockProcessExitError) → throw new
Error(...)`, which means the test will surface as a confusing
"process.exit called with [emitFeedback calls...]" failure when what
the reader actually wants is the underlying error. The test also
asserts `resumedSessionData?.filePath).toContain(sessionId.slice(0, 8))`
which is an *implementation* detail (the 8-char prefix is a property
of the file-naming convention, not a contract); a future refactor that
uses 12 chars or a different separator breaks the test for no
behavioral reason. Tighten to assert the file *exists*, contains the
right `sessionId`, and lives under `chatsDir` — not the prefix
length.

**(3) The import branch's error path calls `runExitCleanup()` then
`process.exit(ExitCodes.FATAL_INPUT_ERROR)` for any failure inside
the try block.** That includes `loadConversationRecord` returning
falsy *and* `fsPromises.writeFile` failing for transient reasons
(disk full, permission denied on `chatsDir` parent). Hard-exiting on
a transient write failure is fine for a one-shot CLI flag, but the
caught-error message that's surfaced via
`coreEvents.emitFeedback('error', `Error importing session from
file: ${err.message}`)` doesn't include the original file path or
the destination path — operators reading the failure log won't know
whether the input file was missing, malformed, or whether the
*output* directory was un-writable. Either include both paths in the
emitted error string or split the try into "load failed" vs. "write
failed" branches with distinct messages.

## Smaller nits (non-blocking)

- `loadConversationRecord` is dynamically imported via
  `import('@google/gemini-cli-core')` in the test (line 1080) but
  statically imported in `gemini.tsx:38`. The static-import path is
  fine; the dynamic-import in the test is just to enable
  `vi.spyOn(coreModule, 'loadConversationRecord')`. Worth a code
  comment in the test explaining why both forms exist.
- `sessionData.startTime = isoNow; sessionData.lastUpdated = isoNow`
  at `gemini.tsx:227-228` discards the original `startTime` from the
  exported file. That's defensible (the imported session is "starting
  now" from this client's perspective) but means there's no audit
  trail back to when the original conversation began — consider
  preserving the original under
  `originalStartTime`/`originalSessionId` fields or in a sidecar.

The feature shape is right and the file-as-seed design is exactly
what cross-project sharing needs. The blockers are around safety
(write-path validation), test quality, and error reporting, all of
which are mechanical to address.
