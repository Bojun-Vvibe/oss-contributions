# google-gemini/gemini-cli PR #26149 — feat(session): expose runtime identity for external observers

- Repo: `google-gemini/gemini-cli`
- PR: https://github.com/google-gemini/gemini-cli/pull/26149
- Head SHA: `eeaf84f12be3a2ce93e2a49b31c5c611aa565e20`
- State: OPEN, +619/-0 across 5 files

## What it does

Adds a per-session "runtime sidecar" file at `<projectTempDir>/<sessionId>/runtime.json` that records `(pid, sessionId, workDir, hostname, startedAt, geminiCliVersion, schema_version=1)` so external tools (terminal multiplexers, tab managers, IDE integrations, observability daemons) can answer "which CLI session is PID X serving?" without parsing argv (which doesn't carry the session id for fresh launches). Pure additive — no behavior changes for users not consuming the sidecar.

## Specific reads

- `packages/core/src/utils/runtimeStatus.ts:127-148` — the atomic write shape:
  ```ts
  const tempPath = `${target}.tmp.${crypto.randomUUID()}`;
  // ...
  fd = fs.openSync(tempPath, 'wx', 0o600);
  fs.writeSync(fd, content, 0, 'utf8');
  fs.fsyncSync(fd);
  fs.closeSync(fd);
  fd = undefined;
  fs.renameSync(tempPath, target);
  ```
  Canonical tmp + fsync + rename pattern. `'wx'` (`O_CREAT | O_EXCL`) on the temp file is defense-in-depth against a pre-placed symlink at the temp path — `crypto.randomUUID()` already makes collision astronomically unlikely, but `wx` ensures even a malicious symlink-attack via a guessable temp name fails closed. Mode `0o600` is correct for a file containing user-identifying state. The `fd !== undefined` cleanup branch in the catch block correctly handles partial-success states.

- `packages/core/src/utils/runtimeStatus.ts:191-233` — the `readRuntimeStatus` validator:
  ```ts
  if (parsed === null || typeof parsed !== 'object' || Array.isArray(parsed)) return null;
  // ... 7 individual field-by-field type checks, each returning null on mismatch ...
  if (typeof schemaVersion !== 'number' || !Number.isInteger(schemaVersion) || schemaVersion !== RUNTIME_STATUS_SCHEMA_VERSION) return null;
  if (typeof pid !== 'number' || !Number.isInteger(pid)) return null;
  // ...
  ```
  Correctly refuses to coerce `null`/array/object into string just to satisfy the type — explicit per the docstring at line 184. The `fatal: true` UTF-8 decode at line 213 also refuses truncated UTF-8. Schema version is matched on equality (`!==`) not range; future schema bumps require either a v1-readers-tolerate-extra-fields rule or a range check, but the docstring at line 196-197 already calls this out: "external consumers should treat unknown fields as forward-compatible additions." This means the **on-disk** schema is forward-compatible but the **reader** is strict — a v2 producer would silently invisibilize itself to v1 readers.

- `packages/core/src/config/storage.ts:367-378` — the new `getSessionTempDir`:
  ```ts
  getSessionTempDir(): string {
    if (!this.sessionId) {
      throw new Error('getSessionTempDir requires a session id; call setSessionId first.');
    }
    return path.join(this.getProjectTempDir(), this.sessionId);
  }
  ```
  Throws on missing-session-id rather than returning a sentinel — correct because the caller in `gemini.tsx:622` is in a try/catch that converts to `debugLogger.debug` (best-effort), so the throw doesn't crash startup. The parallel `getSessionRuntimeStatusPath()` at line 386-388 is a thin wrapper — fine.

- `packages/cli/src/gemini.tsx:614-636` — the call site:
  ```ts
  // Placed after the --list-* / --delete-session early exits so those
  // utility commands don't leave a sidecar claiming a session that never
  // started, and before all three session-serving paths (interactive, ACP,
  // non-interactive). The file is intentionally NOT cleared on exit ...
  try {
    const sessionDir = config.storage.getSessionTempDir();
    await writeRuntimeStatus(sessionDir, {
      sessionId: config.getSessionId(),
      workDir: config.getTargetDir(),
    });
  } catch (err) {
    debugLogger.debug('Failed to write runtime status sidecar:', err);
  }
  ```
  Best-effort try/catch around the write, with a clear comment explaining placement decision. **The "not cleared on exit" decision** (per docstring at line 25-29 and line 31-32) deserves more scrutiny: external observers must verify PID liveness via `kill -0` or equivalent, but the crash-recovery semantics on Windows (where the file may have an open handle even after the writer is gone if AV scanners are involved) and the accumulation rate are not discussed. A long-running shell that opens many sessions to one project will accumulate one `runtime.json` per session-id forever (until the surrounding session-temp-dir is GC'd) — a daemon enumerating these to find live sessions will pay O(historical sessions) per scan.

- Test coverage at `runtimeStatus.test.ts` (304 lines) is comprehensive: round-trips, schema-version mismatch returns null, malformed JSON returns null, type-mismatch on each field returns null, ENOENT returns null, atomic-replace produces second writer's content, hostname/version field round-tripping. **Missing**: a Windows path test (the `'wx'` open mode and rename-onto-existing semantics differ between POSIX and Windows — atomicity on Win32 NTFS rename-replace is technically not guaranteed across volumes), and a test for the file's permission bits (`0o600`) which is mode-relevant on POSIX.

## Verdict: `merge-after-nits`

## Rationale

The feature is well-shaped — atomic write via tmp+fsync+rename, defense-in-depth `wx` open, strict reader that doesn't coerce types, schema versioning, snake_case on-disk for cross-language consumers, and a thoughtful comment block at every decision boundary (placement after early exits, no-cleanup-on-exit, schema-equality matching). The 304-line test surface pins the right invariants. Three nits before merge: (a) the on-disk schema is forward-compatible but the reader uses equality on `schema_version` — call this asymmetry out in the SCHEMA_VERSION constant docstring so a future v2 producer + v1 reader doesn't silently invisibilize sessions; (b) accumulation semantics ("not cleared on exit") deserve a brief note about the GC story or at least a recommendation that external observers cap their scan size by mtime; and (c) the test surface should include either a Windows-skipped-or-conditional test for rename-replace behavior, or an explicit comment that POSIX-only atomicity is the contract.
