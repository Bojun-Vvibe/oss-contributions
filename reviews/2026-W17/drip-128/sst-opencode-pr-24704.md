# sst/opencode #24704 — feat(core): store relative path for sessions

- URL: https://github.com/sst/opencode/pull/24704
- Head SHA: `3ca3849f7b0d69c752c3e06a670bca745ecb3f95`
- Size: +1460 / -4 across 10 files (1419 of additions are migration snapshot)
- Verdict: **merge-after-nits**

## What the change does

Adds an optional `path` column on the `session` SQLite table that stores the
session's CWD **relative to the project worktree**, alongside the existing
absolute `directory` field which stays unchanged.

- Migration:
  `packages/opencode/migration/20260428004200_add_session_path/migration.sql:1`
  is a single `ALTER TABLE session ADD path text;` (nullable, no default) — safe
  additive migration, existing rows null out, no rewrite. Snapshot file at the
  same dir confirms drizzle-kit regenerated metadata cleanly.
- Schema wiring:
  `packages/opencode/src/session/session.ts:77` (`fromRow` reads
  `row.path ?? undefined`), `:101` (`toRow` writes `info.path`), `:163`
  (`Info.path` schema is `optionalOmitUndefined(Schema.String)`), `:255`
  (`UpdatedInfo.path` is `Schema.optional(Schema.NullOr(Schema.String))` — null
  is on-the-wire-clearable, `undefined` is omitted, matching the rest of the
  struct).
- Helper:
  `packages/opencode/src/session/session.ts:129-131`
  `sessionPath(worktree, cwd)` is `path.relative(path.resolve(worktree), cwd)
  .replaceAll("\\", "/")` — the `replaceAll("\\", "/")` is load-bearing for
  Windows so a path stored as `subdir\file` doesn't round-trip differently
  across OSes.
- Layer:
  `:453` accepts `path?: string` on the `create` input and persists it at
  `:463`.
- SDK regeneration at `packages/sdk/js/src/v2/gen/types.gen.ts` and
  `packages/sdk/openapi.json` adds the field on `Session` shape and the
  `UpdatedSession` patch shape.

## What is load-bearing

- The migration is purely additive and nullable — clients on the old code
  reading sessions written by the new code will still decode (the column
  exists in their schema after migrate, just unused), and clients on the new
  code reading old sessions get `undefined` for `path` because of the `??
  undefined` coalesce at `:77`.
- The `optionalOmitUndefined` wrapping at `:163` matches the rest of the
  `Info` struct so JSON output stays compact (no `"path": null` noise) — this
  is the right pattern for a backwards-compatible additive field.

## Nits / questions

1. `sessionPath` is defined at `:129` but I cannot see a call site for it in
   this diff — the code that derives `path` from `(worktree, cwd)` at session
   create time appears to be deferred. Recommend adding at least one caller in
   this PR (e.g. in `Session.create` resolving `input.path ??
   sessionPath(worktree, cwd)`) so the helper isn't dead code on land. If the
   intent is "callers populate `path` themselves for now," then either drop
   the helper from this PR or land a usage test pinning the cross-platform
   slash normalization.
2. No test asserts the cross-platform normalization itself — recommend a unit
   test on `sessionPath("C:\\worktree", "C:\\worktree\\subdir")` returning
   `"subdir"` (forward slash) so a future refactor can't quietly drop the
   `replaceAll`.
3. The `UpdatedInfo` shape allows `path: null` which would clear the column,
   but no test pins the clear-vs-omit semantics. Recommend a test in
   `test/session/session.test.ts` covering: omit-keeps-existing,
   `null`-clears-to-NULL, `string`-overwrites.

## Recommendation

Land after wiring `sessionPath` into the actual `Session.create` path (or
removing it if not needed yet) and adding the cross-platform normalization
test. Migration and schema shape are correct.
