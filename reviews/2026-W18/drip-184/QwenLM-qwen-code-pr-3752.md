---
pr: QwenLM/qwen-code#3752
sha: b232b66df87d500e1d55c4aeca44730f0e66ec07
verdict: merge-after-nits
reviewed_at: 2026-04-30T00:00:00Z
---

# fix(cli): persist directory add entries

URL: https://github.com/QwenLM/qwen-code/pull/3752
Files: `packages/cli/src/ui/commands/directoryCommand.tsx`,
`packages/cli/src/ui/commands/directoryCommand.test.tsx`
Diff: 65+/7-

## Context

`/directory add <path>` only mutated the in-memory `WorkspaceContext`
via `workspaceContext.addDirectory(...)` at
`directoryCommand.tsx:138`, never writing the resolved path back to
`.qwen/settings.json` `context.includeDirectories`. So a user who
added `~/projects/foo` mid-session lost it on restart and had to
re-add — silent state loss. The fix threads the settings service
through the command handler, computes the union of existing +
newly-added directories, and persists via `settings.setValue(
SettingScope.Workspace, 'context.includeDirectories', merged)`.

## What's good

- Persistence block at `directoryCommand.tsx:154-173` runs only when
  `added.length > 0` — the right guard, since a no-op `/directory add`
  (all paths invalid or duplicates of in-memory state) shouldn't
  trigger an unnecessary settings write or churn the file's mtime.
  The `Array.from(new Set([...existing, ...added]))` dedupe at
  `:159-161` correctly hits both cases: (a) the same path passed twice
  in one command, and (b) re-adding a path that was already in
  `context.includeDirectories` from a prior session.
- Shape change at `:140-148`: introduces a single `directory =
  expandHomeDir(pathToAdd.trim())` binding, then uses that binding for
  *both* the in-memory `workspaceContext.addDirectory(directory)` AND
  the persisted `added.push(directory)` list. Prior code at the same
  site did `workspaceContext.addDirectory(expandHomeDir(pathToAdd.trim()))`
  AND `added.push(pathToAdd.trim())` — two different normalisations of
  the same input, so the in-memory state had `~/projects/foo` expanded
  to `/home/user/projects/foo` while the user-facing success message
  showed the un-expanded `~/projects/foo`. The new shape unifies the
  representation, which is also what the persistence layer needs (the
  expanded form is the key the workspace uses elsewhere).
- The downstream `loadServerHierarchicalMemory(...)` call at `:174-183`
  is updated from `[...config.getWorkspaceContext().getDirectories(),
  ...pathsToAdd]` to `[...config.getWorkspaceContext().getDirectories(),
  ...added]`, which means hierarchical-memory loading now sees the
  same expanded-form path list that got persisted, instead of the
  raw user-typed strings. Catches a latent inconsistency where memory
  loading would re-expand `~` and end up duplicating entries that
  the workspace already knew about.
- Error handling at `:163-173` wraps the `settings.setValue` call in
  its own `try` and pushes any failure into the same `errors[]` that
  the command's bottom-of-function summary surface already renders to
  the user via `addItem(MessageType.WARNING, ...)`. Important
  property: a settings-write failure does NOT roll back the in-memory
  `addDirectory` calls — this is the correct choice (the user already
  has working in-memory state for the session, and forcing a
  rollback-on-persist-failure would be a worse UX than "session works,
  reload won't").
- Test additions at `directoryCommand.test.tsx:115-148`:
  - `should persist added directories to workspace settings` (`:115-130`)
    — pre-seeds workspace settings with `existingPath`, runs add for
    `newPath`, asserts `setValue` was called with `[existing, new]` in
    the right order. Locks both the persistence call site and the
    "preserve existing entries" property.
  - `should not duplicate existing workspace settings when persisting`
    (`:132-148`) — re-adds an already-present `existingPath`, asserts
    the persisted array is `[existingPath]` (length 1), which is the
    regression-fence for the dedupe.
- Test fixture additions at `:59-62` extend the mock settings shape
  with `workspace: { settings: {} }` and `setValue: vi.fn()` — the
  exact contract the new code consumes, so the tests don't lie about
  the mock surface.

## Nits

- The dedupe at `:159-161` uses raw string equality via `Set`. Path
  equality on macOS/Linux is case-sensitive but on Windows is
  effectively case-insensitive at the filesystem layer, so adding
  `C:\Foo` after `c:\foo` will (correctly per `Set`) emit two entries
  that point at the same directory and silently bloat
  `context.includeDirectories`. Worth normalising via
  `path.normalize(...).toLowerCase()` on Windows before the Set
  membership check (and a unit test asserting Windows case-insensitivity
  collapsing).
- `settings.workspace.settings.context?.includeDirectories ?? []` at
  `:157-158` reads from `workspace.settings`, but the project elsewhere
  may also read from a `merged` view that layers user + workspace +
  defaults — confirm that writes-to-workspace-only is the correct
  scope for `/directory add` (probably yes, since the path is local to
  the project, but worth a one-line comment naming the intent).
- The error message in `t('Error saving directories to workspace
  settings: {{error}}', ...)` at `:165` interpolates the raw
  `(error as Error).message` — usually fine but if the underlying
  failure is a permission error on `.qwen/settings.json`, the message
  might leak a path the user wasn't looking at. Low risk, but a
  one-line `try { /* sanitise */ }` wrapper would be tidier.
- No test for the *combined* failure path (one path adds successfully,
  another fails workspace validation, persist runs only with the
  successful one). The current tests cover happy-path-add and
  duplicate-add but not the partial-success case that this code
  obviously handles correctly via `added.length > 0` — would be a
  one-test addition that locks the contract.
- The `expandHomeDir` import at `:18` lands as a separate named import
  even though the same module already imports `directoryCommand,
  expandHomeDir` from the same file in tests — fine, but a follow-up
  consolidation pass on the test imports would be tidy.

## Verdict reasoning

Correct fix for a real "feature works in-session, silently doesn't
persist" UX bug. The key shape change is unifying the path
representation between in-memory, persisted, and downstream-consumer
views, which fixes both the bug and a latent inconsistency in the
hierarchical-memory loader. Tests lock the persistence call shape and
the dedupe regression. Nits are about Windows case-insensitivity,
sanitising error message interpolation, adding a partial-success test
case, and a one-line scope comment — none blocking.
