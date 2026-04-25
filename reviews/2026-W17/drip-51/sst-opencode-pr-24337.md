---
pr: 24337
repo: sst/opencode
sha: 7fa7a635d4c63c1ab109217fe8ff8c42a5045199
verdict: merge-as-is
date: 2026-04-26
---

# sst/opencode#24337 — fix(filesystem): tolerate unresolved paths

- **URL**: https://github.com/sst/opencode/pull/24337
- **Author**: pascalandr

## Summary

`AppFileSystem.resolve()` previously caught only `ENOENT` from
`realpathSync()` and rethrew everything else. In practice `realpathSync`
on macOS/Windows can also raise `ELOOP` (symlink cycle), `EACCES`
(component along the path is search-protected), `ENOTDIR` (a parent
became a regular file mid-traversal) and `EPERM`. Any of those would
turn a benign "this path doesn't fully resolve right now" into a hard
exception inside callers that just wanted a normalized form back.

## Reviewable points

- `packages/core/src/filesystem.ts:209-214` — the new body collapses to
  ```ts
  try { return normalizePath(realpathSync(resolved)) }
  catch { return normalizePath(resolved) }
  ```
  i.e. on *any* `realpathSync` failure, fall back to the lexically
  resolved path. This is consistent with how `util/filesystem.ts`
  `normalizePath()` already handles the same call (catch-all,
  fallback to `resolved`), so the two helpers now have matching
  semantics.

- `packages/core/test/filesystem/filesystem.test.ts:325-328` — single
  regression: `resolve(path.join("definitely-missing", randomUUID()))`
  must not throw. Covers the pre-fix behavior path. Adequate as a
  guardrail; broader tests for `ELOOP`/`EACCES` would be hard to
  fixture portably and aren't worth blocking on.

- The catch-all swallows real bugs (e.g. a permission
  misconfiguration that *should* surface) silently. Mitigation: the
  fallback returns the lexically resolved path, not a fabricated one,
  so downstream `stat`/`open` calls will fail with their own
  meaningful errors. That's the right place for the noise.

## Rationale

Two-line change, semantically equivalent to a sibling helper that
already does this, with a regression test. Ship.

## What I learned

When two helpers in the same codebase do "resolve a path, falling
back if not fully realizable," they should agree on the catch
predicate. Diverging here (one `ENOENT`-only, one catch-all) is the
kind of subtle inconsistency that gets papered over by the fact that
both happen to work on the happy path until a user hits `ELOOP` on a
self-referential symlink and only one call site explodes.
