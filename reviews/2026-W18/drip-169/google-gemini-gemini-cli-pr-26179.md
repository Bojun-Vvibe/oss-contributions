# google-gemini/gemini-cli#26179 — `fix(directory): allow removing invalid or unwanted workspace directories`

- PR: https://github.com/google-gemini/gemini-cli/pull/26179
- Head SHA: `f13028fb8110cf88a11da813b026976af2a06faf`
- Author: mini2s

## Summary of change

Adds a `/directory remove <path[,path...]>` slash command companion
to the existing `/directory add`. New subcommand wires through
`packages/cli/src/ui/commands/directoryCommand.tsx`, calling a new
`workspaceContext.removeDirectories(paths)` API that returns
`{removed, notFound}`. Restrictive sandbox blocks the command;
attempting to remove the current `targetDir` (cwd) is rejected with
an error message; partial success (some removed, some not found) is
reported with both INFO and ERROR items. ~180 lines of tests cover
the full matrix.

## Specific observations

- `packages/cli/src/ui/commands/directoryCommand.tsx:275-285` — the
  subcommand metadata sets `autoExecute: false` and
  `showCompletionLoading: false`, matching the `add` sibling. Good
  parity.
- Same file, completion handler (lines 286-313 in the diff): correctly
  derives suggestions from `workspaceContext.getDirectories()` and
  supports the comma-separated multi-path form, mirroring the
  `add` command's UX. The `leadingWhitespace` capture (`lastPart.match(/^\s*/)?.[0]`) preserves user-typed spacing across
  completion — small but nice.
- The cwd protection is in the right place (the command rejects
  removal of `targetDir` *before* calling
  `removeDirectories`). However, the equality check is a string
  compare of `path.resolve(...)` outputs. On macOS this misses the
  `/private/var` ↔ `/var` symlink case, and on case-insensitive
  filesystems it misses `/Users/X/proj` vs `/users/x/proj`. A
  `fs.realpath` + case-fold compare would be more robust; for now
  the test at `directoryCommand.test.tsx:478-495` only exercises
  the trivial-equality path.
- `directoryCommand.test.tsx:496-516` — "should allow removing other
  directories while protecting cwd" asserts that
  `removeDirectories` is called with `[removePath]` only (cwd
  filtered out) AND that an ERROR item is added for the cwd attempt.
  Good — this is the partial-rejection semantics most users will
  hit when they `/directory remove .,./other-thing`.
- Restrictive-sandbox block (lines 460-471): returns an immediate
  `{type: 'message', messageType: 'error', content: ...}` instead
  of the `addItem` path the rest of the command uses. Inconsistent
  but harmless — both surface as visible errors.
- Missing: behaviour when the resolved path *is* in the workspace
  but is currently the *only* remaining writable root. After the
  removal, downstream tools that expect at least one workspace
  directory may misbehave. Worth either a min-1 invariant in
  `workspaceContext.removeDirectories` or a check in the
  command handler. Not visible in this diff which side owns it.
- Test for symlinked / case-folded / trailing-slash paths is absent.
  Should be added before merge given the platform variation.

## Verdict

`merge-after-nits` — solid feature addition with good test surface
for the happy paths. Path-equality robustness (symlinks, case
folding) and the "removed the last writable root" edge case need a
follow-up before this lands as the canonical workspace-management
surface.
