# google-gemini/gemini-cli #26074 — fix(core): handle ENAMETOOLONG/ENOTDIR in robustRealpath

- URL: https://github.com/google-gemini/gemini-cli/pull/26074
- Head SHA: `d31b0158e7310279ed0af2179ea618e69f1f90c9`
- Diff: +48/-14 across `packages/core/src/utils/paths.ts` and `packages/core/src/utils/paths.test.ts`. Closes #26010.

## Context / problem

`robustRealpath` (`packages/core/src/utils/paths.ts`) wraps `fs.realpathSync(p)` and only special-cased `ENOENT` / `EISDIR`. Every other error code was rethrown unhandled. The crash chain users hit:

1. User pastes a JSON blob containing a long hex string or `@email.com`-style token.
2. `atCommandProcessor.parseAllAtCommands` matches it as an `atPath` (regex is intentionally generous).
3. `checkPermissions` calls `resolveToRealPath(path.resolve(targetDir, pathName))`.
4. `robustRealpath` → `fs.realpathSync(p)` → `ENAMETOOLONG` (path component exceeds NAME_MAX/PATH_MAX) → rethrown → unhandled promise rejection in the UI layer → `CRITICAL: Unhandled Promise Rejection` banner crashes the CLI.

`ENOTDIR` is the parallel case: `@path/to/file.json/extra` where `file.json` is a regular file makes `realpathSync` raise `ENOTDIR` on the intermediate component.

## What the fix does

`paths.ts:415-422` extracts a small `hasErrorCode(e: unknown, codes: readonly string[]): boolean` helper — `!e || typeof e !== 'object' || !('code' in e) ? false : typeof e.code === 'string' && codes.includes(e.code)`. Refactors the two existing inline `typeof e === 'object' && 'code' in e && (e.code === 'ENOENT' || e.code === 'EISDIR')` checks to use it (`:432, :443`). Then adds a new branch at `:451-458`:

```ts
if (hasErrorCode(e, ['ENAMETOOLONG', 'ENOTDIR'])) {
  return p;  // Not a resolvable real path — let downstream fileExists/fs.stat reject normally.
}
throw e;
```

Critically, this branch is **separate** from the `ENOENT`/`EISDIR` branch. The PR author correctly identifies that the `ENOENT` branch's symlink-aware parent-walking (`return path.join(robustRealpath(parent, visited), path.basename(p))`) would be **wrong** for `ENAMETOOLONG`/`ENOTDIR` because the basename is garbage in those cases — there's no meaningful parent to walk up to. So the new branch returns `p` unchanged, deferring to downstream `fileExists`/`fs.stat` callers (which already gracefully reject `ENOENT`, see the cited `atCommandProcessor.ts:301`).

## Specific references

- `paths.ts:415-422` — `hasErrorCode` helper. `!e` short-circuit + `typeof === 'object'` + `'code' in e` + `typeof code === 'string' && codes.includes(code)` is the right narrowing chain. `readonly string[]` parameter type prevents accidental mutation by callers.
- `paths.ts:432, :443` — refactor of the two existing `ENOENT`/`EISDIR` checks. Pure refactor, semantics preserved (the PR description explicitly calls this out).
- `paths.ts:451-458` — the new branch. The 5-line code comment naming the intent ("input is not a resolvable real path — for example a caller is probing whether a pasted string is a file reference") is the right level of context for the next maintainer; without it, it would look like a swallowed error.
- `paths.test.ts:564-580` — `should return input path even if fs.realpathSync fails with ENAMETOOLONG`. Mocks `realpathSync` to throw `ENAMETOOLONG`, asserts `resolveToRealPath('/tmp/' + 'a'.repeat(4096))` returns the input unchanged. References the issue number in the test comment.
- `paths.test.ts:583-595` — parallel test for `ENOTDIR`. Same mock pattern, asserts `resolveToRealPath('/tmp/file.json/extra')` returns the input unchanged.

## Risks / nits

- Both new tests use `vi.spyOn(fs, 'realpathSync').mockImplementationOnce(...)` — `mockImplementationOnce` is correct (test isolation), but if a future test in the same describe block expects the real `realpathSync` and runs first under a parallel test runner, ordering could matter. Not a real concern with the `Once` qualifier.
- The `hasErrorCode` helper is defined as a free function in `paths.ts`. If similar narrowing exists in other files (likely — Node fs error codes are pervasive), it's a candidate for a shared util module. Out of scope for this PR.
- The fix is correct only because `atCommandProcessor.ts:301` already handles `ENOENT` from downstream `fileExists`/`fs.stat`. If a future caller of `resolveToRealPath` doesn't tolerate "input returned unchanged when path doesn't exist", they'd get bad data instead of a clear error. The contract change ("returns input unchanged for unresolvable paths") deserves a JSDoc on `resolveToRealPath` itself, not just the inline comment in `robustRealpath`.

## Verdict

`merge-as-is` — root-cause analysis is correct (tracing from paste → at-mention regex → `realpathSync` → `ENAMETOOLONG` → unhandled rejection), the fix is correctly scoped (separate branch from `ENOENT`/`EISDIR` because the parent-walk semantics differ), the helper extraction is a real readability win for the existing inline error-code checks, and both new branches have paired regression tests with the issue number cited inline.
