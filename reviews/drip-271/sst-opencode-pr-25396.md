# sst/opencode #25396 — fix(opencode): replace Expand-Archive with .NET ZipFile on Windows

- URL: https://github.com/sst/opencode/pull/25396
- Head SHA: `a594dd2a3a40e128eb318aba7c116a2a1987b06d`
- Author: @Snakeblack
- Stats: ~50 lines changed across 2 files

## Summary

Drops the `Expand-Archive` cmdlet (which fails to autoload its module
when the host is launched via Bun on Windows, generating noisy
`CouldNotAutoloadMatchingModule` stderr) and replaces both call-sites
with a hand-rolled `System.IO.Compression.ZipFile.OpenRead` loop that
walks entries, creates parent dirs, and writes via `ExtractToFile`.

## Specific feedback

- `packages/opencode/src/util/archive.ts:8-30` — comment correctly
  explains why the 3-arg `ExtractToDirectory(overwrite)` is avoided
  (Core-only) AND why entry-by-entry is preferred (sibling artifacts in
  shared bin dirs would be wiped). That's exactly the kind of context a
  reviewer needs; please keep it.
- `packages/opencode/src/util/archive.ts:18-24` — directory entries are
  detected by `EndsWith('/')`. Real-world zips occasionally use `\` as
  the in-archive separator (older Windows zippers). Worth either
  normalizing (`replace('\\','/')`) before the check or testing both.
- `packages/opencode/src/file/ripgrep.ts:268-279` — same hand-rolled
  loop appears here and in `archive.ts`. Suggest factoring into a single
  `extractZipViaPowerShellDotnet(archive, dir)` helper so the next
  Windows fix only has to land in one place.
- **Zip-slip**: `Join-Path '$dir' $e.FullName` does not strip `..`
  segments. A malicious archive with an entry named `../../etc/x` would
  escape `dir`. `Expand-Archive` had the same flaw, so this PR is not a
  regression — but since you're already rewriting the path, consider
  rejecting entries whose `[System.IO.Path]::GetFullPath($target)` does
  not start with `[System.IO.Path]::GetFullPath('$escapedDir')`.
- Single-quote escaping (`replaceAll("'", "''")`) is the right approach
  for PowerShell single-quoted strings; good.
- No tests added. A round-trip test using a small fixture zip into a
  temp dir would lock in the fix; current changes are platform-gated to
  win32 so CI on Linux/mac won't regress them.

## Verdict

`merge-after-nits` — solves a real Windows-on-Bun pain point with the
right primitive. Refactor the duplicated loop and either fix or
explicitly document the zip-slip behaviour before merge; backslash
directory entries are a nice-to-have.
