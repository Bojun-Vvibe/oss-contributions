# sst/opencode PR #24319 — show linked directories in file list

@d8bae567 · base `main` · +22/-1 · author `pascalandr`

## Summary
Closes #21444. `File.list()` was treating every symlink as a file because `readDir` reports the *link* type, not the target's. On Windows that's especially bad — junctions are dir-symlinks, so the file tree never expanded them. The fix `stat()`s the target when the entry is a symlink and uses the resolved type.

## What changed
- `packages/opencode/src/file/index.ts:598-604` — replaces:
  ```ts
  const type = entry.type === "directory" ? "directory" : "file"
  ```
  with:
  ```ts
  const target =
    entry.type === "symlink" ? yield* appFs.stat(absolute).pipe(Effect.catch(() => Effect.void)) : undefined
  const type = entry.type === "directory" || target?.type === "Directory" ? "directory" : "file"
  ```
- `packages/opencode/test/file/index.test.ts:648-666` — adds `marks linked directories as directories` test that creates a real `fs.symlink(target, link, …)` (with a `"junction"` flavor on win32) and asserts the listed node has `type: "directory"`.

## Key observations
- The narrow `entry.type === "symlink"` guard is the right perf decision — only the symlink path takes the extra `stat()` syscall, regular dir/file entries are unaffected.
- `Effect.catch(() => Effect.void)` swallows broken-symlink errors and falls through to `"file"` — that's pragmatic, but worth a comment because at first read it looks like the error is being lost. A broken dangling symlink showing as `"file"` is debatably the wrong default; arguably it should be omitted from the listing entirely or annotated. Not blocking.
- The test guards Windows symlink-permission failures with `if (!(await Filesystem.exists(link))) return` — that's a no-op on machines without the privilege. Reasonable, but it means the win32 path has effectively no enforcement on CI runners that lack the permission. A comment explaining the soft-skip would help.
- Test exercises symlink-to-dir but not symlink-to-file or broken-symlink. Three cases would be ideal; one is acceptable for a bug-fix PR.

## Risks/nits
- Each symlink encountered now incurs a `stat()`. For directories with many symlinks this is a perf regression of one syscall per link. Not a concern for typical repos.
- `target?.type === "Directory"` capitalization is correct for `appFs.stat`'s return — confirm by reading the FS effect typings.

**Verdict: merge-after-nits**
