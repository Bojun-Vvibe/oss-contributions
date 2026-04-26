# sst/opencode #24359 — fix(app): show project sessions across worktrees

- **Author:** pascalandr
- **Head SHA:** `7fd7994ee04b3e5aeb1f5db3eec6a703e04a0962`
- **Size:** +45 / -15 (5 files)
- **URL:** https://github.com/sst/opencode/pull/24359

## Summary

The app session loader unconditionally sent `directory` to
`session.list`, so the project-level session sidebar was filtering
sessions to a single worktree even though all worktrees of a given
project share the same `project_id`. Result: switching to a different
worktree hid sessions started in others. The fix introduces an opt-out
`filterDirectory` flag that defaults to `true` (preserving existing
workspace-scoped views) but is set to `false` for the top-level project
session view.

Closes #23594; related to #14082 and #17739.

## Specific findings

- **The wire-level fix is correct.**
  `packages/app/src/context/global-sync/session-load.ts:5-9` builds the
  query lazily and conditionally omits `directory`:

  ```ts
  const query = (limit?: number) => ({
    ...(input.filterDirectory === false ? {} : { directory: input.directory }),
    roots: true as const,
    ...(limit ? { limit } : {}),
  })
  ```

  The strict `=== false` check (rather than truthy check) is the right
  shape because the field is optional and defaulting to `true` for
  back-compat is the documented behaviour. Both the limited
  (`query(input.limit)`) and fallback (`query()`) paths are routed
  through the same builder, so there's no risk of one branch silently
  retaining the directory while the other drops it.

- **Cache key invalidation is the second important change.**
  `packages/app/src/context/global-sync.tsx:43-46` adds a third element
  to the queryKey: `[directory, "loadSessions", filterDirectory]`. The
  old key was `[directory, "loadSessions"]`, which meant the *same*
  cache slot was being reused for the directory-scoped and
  project-wide loads — a previously-loaded directory-scoped result
  would be returned for a project-wide query. The new key separates
  them. The `sessionMeta` map is similarly extended at
  `global-sync.tsx:58, 159` to track `{filterDirectory, limit}`
  together, and the dedup gate at line 159 now requires both to match
  before reusing a cached slot. This is the kind of subtle cache-key
  change that's easy to miss; good to see it caught and tested.

- **The new test is well-targeted.**
  `global-sync.test.ts:65-79` asserts that a `filterDirectory: false`
  call produces a single `{roots: true, limit: 10}` query — no
  `directory` field. The two existing tests (lines 27-46) keep their
  default-true behaviour pinned, so this PR can't silently regress
  the workspace-scoped path.

- **Minor type-shape change has consumer ripple.**
  `types.ts:121-124` changes `RootLoadArgs.list`'s query parameter from
  `{ directory: string; ... }` to `{ directory?: string; ... }`. Any
  in-tree caller that destructured `directory` from the query arg now
  has to handle the `undefined` case. The PR diff doesn't touch any
  such caller, which is consistent with the test fixture also being
  updated to optional, but worth a `grep` audit before merge —
  TypeScript should catch any miss but the diff would be more
  reassuring with the audit results in the description.

- **`layout.tsx` change is the actual call-site adoption.** Lines
  1827-1828 in the layout (only partially visible in the diff window)
  presumably toggle `filterDirectory: false` for the project-level view
  while leaving it default for the workspace view. Worth verifying the
  branching is keyed off the right state (e.g. "currently in project
  view" vs "an explicit workspace is selected") and not a more brittle
  proxy.

- **Banned-string check:** title and diff are clean.

## Verdict

`merge-after-nits`

(One-line confirmation in the PR body that all in-tree consumers of
`RootLoadArgs.list` handle the now-optional `directory`, and a brief
note describing the layout.tsx call-site logic.)
