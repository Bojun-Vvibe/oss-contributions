# QwenLM/qwen-code PR #3714 — feat(core): write runtime.json sidecar for active sessions

- URL: https://github.com/QwenLM/qwen-code/pull/3714
- Head SHA: 098cf04bb51c
- Files: `packages/cli/src/gemini.tsx`, `packages/core/src/config/storage.ts`, `packages/core/src/index.ts`, `packages/core/src/utils/runtimeStatus.ts` (new), `packages/core/src/utils/runtimeStatus.test.ts` (new)
- Verdict: **merge-after-nits**

## Context

External tooling (terminal multiplexers, IDE integrations,
status daemons) currently has no first-class way to map a
running CLI process back to its session id and working directory.
This PR introduces a small `runtime.json` sidecar that the CLI
writes alongside its chat log on startup, plus
`writeRuntimeStatus` / `readRuntimeStatus` / `clearRuntimeStatus`
helpers and a `RUNTIME_STATUS_SCHEMA_VERSION` for forward-compat.

## Design analysis

The integration point in `gemini.tsx:174-191` is appropriately
guarded:

```
try {
  const sessionId = config.getSessionId();
  const runtimeStatusPath = config.storage.getRuntimeStatusPath(sessionId);
  await writeRuntimeStatus(runtimeStatusPath, {
    sessionId,
    workDir: config.getTargetDir(),
    qwenVersion: version,
  });
} catch {
  // ignored: best-effort, never block UI startup.
}
```

The "swallow all errors" pattern is correct here — a read-only
filesystem, a missing parent dir, or a sandbox restriction must
never prevent the UI from starting. The cost is that genuine
bugs (e.g. malformed paths, permission misconfiguration) become
invisible. A `tracing` / `console.debug` log inside the catch
would let users debugging "why isn't my multiplexer seeing the
session" find out cheaply, without surfacing noise to normal
users.

The path layout in `storage.ts:233-245` co-locates the sidecar
with the chat log:
`<projectDir>/chats/<sessionId>.runtime.json`. This is good —
external observers already need to know the chats dir to scan
sessions, so they don't need a second discovery mechanism. The
docstring spells this out.

The sidecar field naming uses snake_case on disk
(`session_id`, `work_dir`) per the test expectations at lines
118-120. That's a reasonable convention for a JSON file consumed
by external Unix tooling, but worth confirming the project style
guide doesn't standardize on camelCase across the wire — a quick
grep of other JSON sidecars in the repo would settle it.

## Risks / suggestions

1. **No `clearRuntimeStatus` invocation visible in this diff** for
   the shutdown path. If the file is created on startup but never
   removed on clean exit, observers will see stale entries
   pointing at dead PIDs. Either (a) register an exit handler
   that calls `clearRuntimeStatus`, or (b) document that
   observers must validate `pid` liveness before trusting the
   sidecar. The latter is more robust against crashes; the
   former is nicer in the happy path. Both is best.
2. The sidecar exports a schema version constant
   (`RUNTIME_STATUS_SCHEMA_VERSION`) — good. Make sure the
   reader actually checks it and degrades gracefully on
   mismatch, rather than blindly destructuring fields.
3. PID is included (per the test). Since the sidecar lives under
   the user's own home dir tree, this is fine, but document
   that the file may leak the working directory and CLI version
   to anything that can read the project chats dir. Probably a
   non-issue but worth a one-liner.
4. The 522-line PR is mostly the new helper module + tests
   (241 lines for the test file alone). The test coverage looks
   thorough — round-trip, missing fields, clear, schema-version,
   etc. — which is what you want for code that other tools will
   be parsing for years.

## What I learned

Sidecar files for "what's running where" are an old pattern
(systemd `.pid`, `nohup.out`, etc.) that's still the right
answer when you want low-friction interop with external tools.
The two failure modes to design against are (a) startup
filesystem flakiness — handle with try/swallow + best-effort —
and (b) stale entries from crashes — handle with PID liveness
checks on the reader side. This PR addresses (a) cleanly; (b)
is the missing half.
