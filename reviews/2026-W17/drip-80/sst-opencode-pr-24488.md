# sst/opencode PR #24488 — fix(opencode): avoid PowerShell for zip extraction

- **PR:** https://github.com/sst/opencode/pull/24488
- **Author:** mugnimaestra
- **Head SHA:** `fc643f69846dc095b34c7ac1dd2431ac4e9440f3`
- **Files:** 3 (+82 / -21)
- **Verdict:** `merge-after-nits`

## What it does

Fixes #24489: on Windows, ripgrep bootstrap could fail because the previous code
shelled out to `Expand-Archive` from PowerShell, which depends on the
PowerShell archive module autoloading correctly. On stripped
PowerShell installs (or in environments with `$PSModulePath` tampering) skill
loading would die silently before any rg-based search worked.

The fix swaps PowerShell extraction for the existing in-process
`@zip.js/zip.js` dependency, routes ripgrep ZIP extraction through the shared
`Archive.extractZip` helper, and adds zip-slip protection on entry paths.

## Specific reads

- `packages/opencode/src/file/ripgrep.ts:253-258` — old code path constructed a
  `powershell.exe`/`pwsh.exe` invocation with single-quote escaping
  (`archive.replaceAll("'", "''")`) and `$global:ProgressPreference =
  'SilentlyContinue'`. New code is six lines wrapping `Archive.extractZip`
  in `Effect.tryPromise` with proper error coercion. Net win on
  cross-platform consistency — same code path now on Windows / Linux / macOS
  for the `.zip` branch.
- `packages/opencode/src/util/archive.ts:5-13` — new `destination(root,
  filename)` helper. Normalizes `\` → `/` (necessary because zip entry names
  on Windows-authored archives often contain backslashes), resolves against
  root, and rejects entries where the relative path is empty, starts with
  `..`, or is absolute. **This is the right shape for zip-slip defense** —
  the three-condition check matches what e.g. Go's `archive/zip` examples
  recommend. One nit: the rejection happens per-entry but the partial
  extraction up to that point is left on disk. For a hostile archive that's
  fine (you fail loudly), but a `try { … } catch { rmdir(root) }` outer
  cleanup would make recovery cleaner.
- `packages/opencode/src/util/archive.ts:18-32` — main loop. Uses
  `Bun.file(zipPath)` + `BlobReader` + `ZipReader` with
  `useWebWorkers: false` (correct for a CLI — no worker pool to manage). For
  each entry, mkdir-recursive the parent, then `Bun.write(target, await
  entry.getData(new Uint8ArrayWriter(), …))`. Restores executable bit on
  non-Windows when `entry.executable` is set — important for the rg binary
  which needs `0o755`.
- The old `extractZip` had a non-Windows fallback to `unzip -o -q` (line 13
  pre-diff). That fallback is **gone**. Now every platform uses zip.js. This
  is the right call (consistent behavior, no dependency on `unzip(1)`
  presence on minimal Linux containers), but it's worth flagging in the PR
  description because anyone running on a system where `unzip` was the only
  reliable extractor will now go through Bun. CI tests cover this, per the
  PR body (`bun test test/util/archive.test.ts`).
- `entry.getData` is awaited in serial inside the for loop. For large
  archives (thousands of entries) this could be slower than the old `unzip`
  binary. Ripgrep ZIP is small (one binary + readme), so not a real concern
  here, but a future caller of `Archive.extractZip` on a larger archive
  should know.

## Risk surface

**Low.** Self-contained refactor, replaces a known-flaky shell dependency
with an in-process library that's already in the dep tree. Zip-slip defense
is added (was absent in both old paths — the old `Expand-Archive` and old
`unzip -o` were both vulnerable to crafted archives).

Three small risks:

1. **Symlink entries** — zip.js sets `entry.directory` for dirs and exposes
   `executable`, but I don't see explicit handling of symlink-type entries
   (`unix_external_attributes & 0xA000`). If a malicious archive contains a
   symlink pointing outside `root`, the current `destination()` check on the
   filename catches the symlink *path* but a subsequent file write through
   the symlink could still escape. ripgrep's archive doesn't have symlinks,
   so practically fine, but worth a follow-up if `Archive.extractZip` gets
   used for arbitrary archives.
2. **Performance regression** — serial `await entry.getData()` per file. Not
   measurable for the rg archive (one binary).
3. **Error type loss** — `catch: (cause) => (cause instanceof Error ? cause
   : new Error(String(cause)))` flattens the original error chain. If
   `Archive.extractZip` throws a structured zip-format error, downstream
   logging will only see the message string. Consider preserving the cause
   via `new Error(message, { cause })`.

## Suggestions before merge

- Verified via the test list in the PR body and the Windows manual smoke
  test ("deleted cached `rg.exe`, confirmed Skill 'context7' loads"). That's
  enough for landing.
- Nit (non-blocking): preserve `cause` on the error wrap.
- Nit (non-blocking): document in `archive.ts` that this helper does not
  currently handle symlink entries, so future callers know not to pass
  untrusted archives until that's added.

Verdict: merge-after-nits — replaces a fragile shell dependency with a
sound in-process implementation, adds the zip-slip check that was missing in
both legacy paths, and the tests cover the happy path. Symlink-handling and
error-cause preservation are reasonable follow-ups, not blockers.

## What I learned

`Expand-Archive` depends on a PowerShell module that ships with the PS
install but can be unloaded or path-broken in stripped/customized PS
deployments — exactly the kind of environment Windows users with locked-down
PS profiles end up in. Shelling out to a "standard" platform tool isn't a
zero-cost portability win; it transfers the failure mode from your code to
the platform's module-loader, where it's harder to diagnose. In-process
extraction trades a small performance loss for one fewer system dependency.
