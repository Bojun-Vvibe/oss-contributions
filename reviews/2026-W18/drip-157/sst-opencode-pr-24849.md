# sst/opencode PR #24849 — feat(core): filter sessions by path and add setting to disable

- **PR**: https://github.com/sst/opencode/pull/24849
- **Author**: jlongster
- **Merged**: 2026-04-28T20:49:13Z
- **Head SHA**: `155daf6484a1`
- **Size**: +360/-27 across 10 files
- **Verdict**: `merge-after-nits`

## Context

Until this PR, the TUI's session list pulled "all sessions for the current
worktree" via the existing `directory` query parameter on `GET /session`.
For developers who keep a top-level monorepo open in one workspace and bounce
between subpackages, this was either too narrow (the `directory`-equality
filter at `session.ts:776-778` matched only the exact cwd) or too broad
(scope=workspace returns sessions from sibling packages they don't care
about). This PR adds a third axis: *path-prefix* filtering keyed off the
session's recorded `path` (workspace-relative), exposed as a new
`?path=<rel>` query and a TUI toggle (`session_directory_filter_enabled`,
default `true`).

## What changed

1. **HTTP API surface** (`packages/opencode/src/server/routes/instance/httpapi/session.ts:46-48`,
   and the legacy `instance/session.ts:65-69`): the `ListQuery` schema gains
   two optional fields — `scope?: "project"` and `path?: string` — both
   forwarded to `Session.list`. The legacy route at
   `instance/session.ts:80-84` adds `directory: query.scope === "project" ?
   undefined : query.directory`, which is the right way to model the
   mutual-exclusion: when the caller asks for the project-wide scope, the
   path-equality filter is intentionally dropped.

2. **DB query** (`packages/opencode/src/session/session.ts:761-783`): the new
   `path` branch builds an `or(eq(SessionTable.path, input.path), like(SessionTable.path, "${input.path}/%"))`
   condition — i.e. exact match OR descendant prefix on `/`-bounded path
   segments. The fallback for legacy rows where `SessionTable.path IS NULL`
   keeps the old `directory`-equality filter, gated on `input.directory`
   being set, so the migration is non-disruptive.

3. **TUI plumbing** (`packages/opencode/src/cli/cmd/tui/context/sync.tsx:113-130`):
   a new `sessionListQuery()` helper computes the workspace-relative path
   from `project.data.instance.path.{worktree,directory}` via
   `path.relative(...).replaceAll("\\", "/")`, gated on the
   `session_directory_filter_enabled` KV flag (default `true`). When the
   flag is off OR the project doesn't expose a worktree+directory pair, it
   falls back to `{ scope: "project" }`. The list is then refetched in
   `dialog-session-list.tsx:35-41` whenever either `search()` or
   `sync.session.query()` changes (the latter is now a tracked signal so the
   toggle propagates).

4. **TUI toggle UX** (`packages/opencode/src/cli/cmd/tui/app.tsx:739-751`):
   adds a `Disable session directory filtering` / `Enable session directory
   filtering` entry to the System command palette. Toggle calls
   `kv.set(...)` then `await sync.session.refresh()`, which is the right
   ordering — flip the flag first so the next `listSessions()` call picks
   up the new query shape.

5. **Test coverage** (`packages/opencode/test/cli/cmd/tui/sync.test.tsx`,
   new file, 149 lines): mounts the full `<ArgsProvider> → <ExitProvider> →
   <KVProvider> → <SDKProvider> → <ProjectProvider> → <SyncProvider>` stack
   with a fake `fetch` that records every `/session` URL hit, then asserts
   the recorded `path=` and `scope=` query params under
   filter-enabled vs filter-disabled.

## Why `merge-after-nits`

The semantics are right and the test that mounts the entire context stack is
exactly the kind of integration test you want for a TUI signal-graph change
like this — but a few sharp edges:

- **Windows-path collapse**: `path.relative(...).replaceAll("\\", "/")` is
  the correct normalization for the wire format, but it doesn't normalize
  drive letters or UNC paths, and it doesn't reject the `..` case. If the
  worktree resolves to one drive and the directory to another (rare but
  possible with junction points), `path.relative` returns an absolute path
  starting with the drive letter, which then gets shipped as a `path=`
  query param and silently matches nothing. Worth adding a guard:
  `if (rel.startsWith("..") || path.isAbsolute(rel)) return { scope: "project" }`.

- **Empty-string ambiguity at the root**: when `path.relative(worktree,
  worktree)` returns `""`, the new branch in `session.ts:763` does
  `if (input.path) { ... }` so the empty-string falls through. The
  intended semantics at the worktree root are "match all sessions in this
  worktree" — currently the code falls into the `else if (input?.scope !==
  "project" && !Flag.OPENCODE_EXPERIMENTAL_WORKSPACES)` branch with
  `input.directory` undefined, so no session-filter is added at all. That
  may be correct (worktree root = list everything) but it should be
  asserted by a test, not inferred from "the truthy check accidentally did
  the right thing."

- **`refresh()` vs `createResource` recompute race**: the System-palette
  toggle calls `kv.set(...)` then `await sync.session.refresh()`, but
  `dialog-session-list.tsx`'s `createResource(() => ({ query: search(),
  filter: sync.session.query() }))` *also* recomputes when the KV signal
  fires. Both paths will end up issuing a `/session` request — the one
  from `refresh()` and the one from the resource recomputation. Not
  harmful (the second response just overwrites the first), but a clear
  comment near the toggle handler explaining the double-fetch would help
  the next reader.

- **Legacy `directory`-fallback inside the path branch**: the construction
  at `session.ts:766-770` —
  ```ts
  input.directory
    ? or(...conds, and(isNull(SessionTable.path), eq(SessionTable.directory, input.directory))!)!
    : or(...conds)!
  ```
  is doing two things at once: prefix-matching new sessions with `path` set
  AND falling back to the old `directory` filter for sessions migrated
  before `path` started being recorded. That fallback is silently scoped to
  "only when caller passed both `path` and `directory`" — a TUI caller
  doesn't (the `sessionListQuery()` helper builds `{ path }` only) — so
  the legacy-row safety net never triggers from the TUI path. If the
  intent is "TUI users with a mix of pre- and post-migration sessions
  still see the old ones," the helper needs to also pass `directory:
  project.data.instance.path.directory`.

## What I learned

When a list-API gains a new dimension (here: `path`) alongside an existing
one (`directory`) that the new dimension partially supersedes, the
right-shaped contract in the *route handler* is mutual exclusion driven by
an explicit `scope` enum, exactly as `instance/session.ts:80` does
(`directory: query.scope === "project" ? undefined : query.directory`) — but
it's easy to forget to apply the same mutual-exclusion logic at the SQL
layer. This PR gets the route layer right and gets the SQL layer mostly
right, but the legacy-row fallback inside the `path` branch ends up
load-bearing on an extra arg the callers don't pass. The lesson: when you
add a new filter dimension, write the truth table for `(directory, path,
scope)` first — including the legacy-data-shape rows — and pin each cell
with a query test, before you let the new field land in the schema.
