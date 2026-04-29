# google-gemini/gemini-cli#26199 — fix(cli): sweep all projects for session retention

- **Repo:** google-gemini/gemini-cli
- **PR:** #26199
- **Head SHA:** fda82f85fe32e14c76f0004807950fd377308932
- **Base:** main
- **Author:** community

## Summary

Expands session-retention cleanup from "current project's temp dir only" to "every project temp dir under the global temp root", so a user who runs the CLI in project A doesn't accumulate stale sessions in project B forever. Introduces `getCleanupRoots(projectTempDir)` at `packages/cli/src/utils/sessionCleanup.ts:79-101` which checks whether the given project temp dir lives under `Storage.getGlobalTempDir()` — if yes, it enumerates *every* sibling subdir; if no (test/custom paths), it falls back to single-root behavior. Carries `tempDir` and `chatsDir` through the `SessionFileEntry` pipeline so per-file deletion targets the correct root, and changes the `processedShortIds` dedupe key from `shortId` alone to `${chatsDir}:${shortId}` to prevent cross-project short-ID collisions from suppressing a deletion.

## File:line references

- `packages/cli/src/utils/sessionCleanup.ts:49-56` — new `CleanupRoot` and `CleanupSessionFileEntry` types
- `packages/cli/src/utils/sessionCleanup.ts:69-77` — `cleanupSessionAndSubagentsAsync` now takes `tempDir: string` directly instead of `config: Config` (decouples per-root deletion from current-project assumption)
- `packages/cli/src/utils/sessionCleanup.ts:79-101` — `getCleanupRoots()` with the under-global-temp-dir guard via `path.relative` + `!startsWith('..')` + `!path.isAbsolute(rel)` triple-check
- `packages/cli/src/utils/sessionCleanup.ts:147-161` — parallel `Promise.all` per-root file collection, flattened into one `allFiles`
- `packages/cli/src/utils/sessionCleanup.ts:184-186` — dedupe key changed to `${chatsDir}:${shortId}` (prevents cross-project collision)
- `packages/cli/src/utils/sessionCleanup.ts:200-212` — matching-files filter now filters by `f.chatsDir === sessionToDelete.chatsDir` so deletion is scoped to the file's own root
- `packages/cli/src/utils/sessionCleanup.ts:268-274` — `identifySessionsToDelete` made generic over `T extends SessionFileEntry`
- `packages/cli/src/utils/sessionCleanup.test.ts:283-360` — new test "should apply retention across all project temp directories" with `Storage.getGlobalTempDir` mock + two project subdirs

## Verdict: **merge-after-nits**

## Rationale

The bug is real (per-project retention silently growing unboundedly across projects) and the fix's architectural choice — scope-detect via `path.relative` and degrade gracefully — is the right call for testability. The new test exercises both the scan path and the per-root deletion path, including the subagent-logs cleanup. Concerns:

1. **`Promise.all` over potentially many project dirs has no concurrency cap.** `sessionCleanup.ts:158` reads N project dirs in parallel; a long-tail user with 200+ stale project temp dirs (CI, monorepo branch worktrees) will fan out 200 parallel `readdir` calls on every CLI startup. Wrap in a small concurrency limiter (4-8) or document the fan-out risk.
2. **The `getCleanupRoots` fallback at `sessionCleanup.ts:88-90` is fragile on Windows.** `path.relative` on Windows can return values that start with `..` for case-sensitive disagreement (e.g., `c:\Temp` vs `C:\Temp`). Add a Windows-specific test case proving the fallback fires correctly when the project temp dir is canonically under global temp but case-mismatched.
3. **No upper bound on how many project temp dirs are scanned.** A user who has been on the CLI for 2 years could have hundreds of project hashes under `~/.gemini/tmp/`. Consider a soft cap with a debug-log message ("scanned 250 project temp dirs in 800ms — consider running `gemini sessions prune-empty`").
4. **The new test at `sessionCleanup.test.ts:283-360` only covers the happy path (one current, one stale).** Add a case where the *current* session in a *different* project temp dir would otherwise be eligible for deletion — `getSessionId()` lives on `config` and is global, so a stale session in project B with the same short-ID as the active session in project A could now be deleted (or not) depending on dedupe key ordering. Lock that semantics down with an explicit assertion.
5. **`cleanupSessionAndSubagentsAsync` signature change from `config: Config` to `tempDir: string`** is a clean simplification but worth a one-line CHANGELOG note since this is an exported helper that downstream test fixtures may have been calling.
