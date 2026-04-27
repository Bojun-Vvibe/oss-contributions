# google-gemini/gemini-cli #25950 — fix: prevent false command conflicts when launching from home directory

- **Repo**: google-gemini/gemini-cli
- **PR**: #25950
- **Author**: stbenjam
- **Head SHA**: a4e99cbc85f04d3b12c7390c85e5b56dc1960c0b
- **Size**: +44 / −8 across two files (one prod, one test).

## What it changes

Five-line behavior fix in
`packages/cli/src/services/FileCommandLoader.ts:152-175`
plus a 28-line regression test pinning the symptom.

The bug: `FileCommandLoader.loadCommands` builds a list
of directories to scan for `.toml` command files. The
first entry is "user commands"
(`Storage.getUserCommandsDir()`, typically
`~/.gemini/commands/`) and the second is "project
commands" (`storage.getProjectCommandsDir()`, typically
`<projectRoot>/.gemini/commands/`). When the user
launches gemini-cli with `cwd === $HOME`, both
directories resolve to the *same* path
(`~/.gemini/commands/`), and the loader scans it twice
— once tagged `USER_FILE`, once tagged `WORKSPACE_FILE`
— producing a phantom "command conflict" warning for
every command the user has defined and shadowing each
user-file command with a duplicate workspace-file
entry.

The fix introduces a short-circuit:

```ts
const userCommandsDir = Storage.getUserCommandsDir();
dirs.push({ path: userCommandsDir, kind: CommandKind.USER_FILE });

// 2. Project commands (skip if same directory as user commands, e.g. when
//    cwd is the user's home directory, to avoid false conflict warnings)
if (!storage.isWorkspaceHomeDir()) {
  dirs.push({
    path: storage.getProjectCommandsDir(),
    kind: CommandKind.WORKSPACE_FILE,
  });
}
```

The `storage.isWorkspaceHomeDir()` predicate is presumably
already defined on `Storage` (the diff doesn't add it,
just consumes it), and the call site is the natural
gate.

## Strengths

- **Right minimal fix at the right layer.** The
  alternative would be to deduplicate downstream after
  the scan (e.g. drop workspace-file entries whose path
  resolves to the user-file dir), but that pushes the
  policy decision into the conflict-detection code path
  where it's far from the cause. Skipping the scan
  altogether at the source means no phantom files are
  ever surfaced and the conflict-detection logic stays
  simple.
- **Clear two-line comment** at `:163-164` that names
  the failure mode by its symptom ("false conflict
  warnings") and the trigger condition ("cwd is the
  user's home directory"). A future contributor reading
  this file will understand *why* the gate is there
  without having to dig through git blame.
- **Test pins the canonical symptom** at
  `FileCommandLoader.test.ts:317-344`:
  - Mocks `getProjectRoot` to return `homedir()`
  - Sets up a fake `~/.gemini/commands/` with two
    `.toml` files
  - Asserts `commands.length === 2` (not 4) and that
    `commands.every(c => c.kind === 'user-file')` —
    i.e. no workspace-file duplicates
  - The assertion `expect(commands.every((c) => c.kind === 'user-file')).toBe(true)`
    locks the contract that commands found via this
    scan are tagged as user commands, not as workspace
    commands. That's the substantive correctness
    statement.

## Concerns / nits

- **`isWorkspaceHomeDir()` is consumed but not shown
  in the diff.** The PR assumes the helper exists on
  `Storage`. If this method is added in a sibling
  PR or if the implementation is `path.resolve(this.projectRoot) === path.resolve(homedir())`,
  the comparison needs to be path-normalization-safe
  (`fs.realpathSync` for symlinked home dirs,
  case-insensitive on macOS HFS+/APFS-default, etc.)
  Worth a one-line note in the PR body confirming the
  predicate handles symlinks and case sensitivity, or
  a follow-up test for `cwd = /Users/bob` vs
  `cwd = /private/var/.../bob` (macOS realpath).
- **No coverage of the symlinked-home case.** If a
  user's `$HOME` is symlinked (e.g. `/home/bob ->
  /mnt/users/bob`), the test as written would pass
  even if `isWorkspaceHomeDir` did naive string
  comparison and missed the symlink. A test with a
  symlink fixture would catch that.
- **`isWorkspaceHomeDir` short-circuits for the entire
  workspace.** This is correct for the conflict-warning
  bug, but it means a user who *intentionally* has a
  `~/.gemini/commands/` (user) and a
  `~/some-project/.gemini/commands/` (workspace) and
  then `cd ~ && gemini` loses the workspace-file
  scan — which is the only way to load the *workspace*
  variants of commands. Since `cwd === $HOME` and the
  workspace dir resolves to the user dir, there is no
  workspace dir to skip in this case. So the gate is
  semantically a no-op when there's nothing to skip,
  and the comment correctly identifies "false
  conflict" not "lost commands" as the symptom. Still
  worth a one-line PR body note that "intentional
  workspace at $HOME" is not a supported scenario.
- **Import-cleanup miss.** The diff at
  `FileCommandLoader.test.ts:9` adds `homedir` to the
  import from `@google/gemini-cli-core` — which assumes
  that package re-exports `homedir`. The diff doesn't
  show the export side. If `@google/gemini-cli-core`
  doesn't re-export it, the import will fail; if it
  does, this is fine. Worth a quick `rg "export.*homedir" packages/core/`
  before merge.

## Verdict

**merge-after-nits.** Correct minimal fix at the right
layer, with a regression test that pins both the
deduplication count *and* the kind-tagging. Want
confirmation that `isWorkspaceHomeDir()` exists with
symlink/realpath safety, that `homedir` is actually
re-exported from `@google/gemini-cli-core`, and ideally
a symlinked-home test fixture before merge.

## What I learned

The "two configured directories that resolve to the same
filesystem path" failure mode is a classic config-loader
trap — every loader that has a layered config
(`/etc/foo`, `~/.foo`, `./foo`) eventually trips it when
a user runs from `/etc` or `~`. The right fix is always
to deduplicate at the *source* (skip scanning a dir
whose resolved path equals an earlier-scanned dir),
never at the consumer (drop later-found entries with
matching keys), because the source-dedup also avoids
the phantom-conflict-warning symptom. Worth keeping the
"resolved-path-set membership check before push" pattern
in the toolbox for the next layered config loader.
