---
pr: 24321
repo: sst/opencode
sha: 9f965909344a38d50d920ef6f818495e788dc52c
verdict: merge-as-is
date: 2026-04-27
---

# sst/opencode #24321 — fix(shell): detect Windows bash from PATH

- **Head SHA**: `9f965909344a38d50d920ef6f818495e788dc52c`
- **URL**: https://github.com/sst/opencode/pull/24321
- **Size**: small (120/2 across 2 files; ~2/3 of the diff is tests)

## Summary
Fixes Windows bash discovery so the resolver checks PATH first, also tolerates MSYS2/UCRT64 git layouts that don't have a sibling `..\..\bin\bash.exe`, and explicitly excludes the WSL `bash.exe` that ships under `%SystemRoot%\System32\bash.exe` (which was being silently picked up and produced confusing path translation errors).

## Specific findings
- `packages/opencode/src/shell/shell.ts:72-78` — early `pathbash()` check before the git-derived candidates. Order is right: a user-installed `bash.exe` on PATH should win over the bundled Git-for-Windows one. The fallback list at `:79-83` adds `path.join(git, "..", "bash.exe")` and the `..\..\..\usr\bin\bash.exe` MSYS2 path; both are real layouts seen in the wild and the search is short-circuit `find()` so cost is negligible.
- `shell.ts:86-91` — `pathbash()` strips wrapping quotes from PATH entries with `dir.replace(/^"(.*)"$/, "$1")`. Good — Windows PATH entries with spaces sometimes ship quoted in the env. The `.filter(Boolean)` correctly drops empty entries from a trailing `;`.
- `shell.ts:101-105` — `wslbash()` defends specifically against `%WINDIR%\System32\bash.exe` and the Sysnative WoW64 alias. The case-insensitive comparison via `Filesystem.windowsPath(...).toLowerCase()` is correct for NTFS and matches what `which` would resolve. Worth noting: this only excludes the OS-shipped WSL launcher, not a user-installed `wsl.exe`-wrapped bash on PATH — that's intentional and correct (if the user explicitly puts WSL bash on PATH, that's a deliberate choice).
- `packages/opencode/test/shell/shell.test.ts:90-152` — three good tests: PATH-vs-git precedence, MSYS2 UCRT64 layout, and WSL exclusion. Each uses `withWindowsEnv()` to scope env mutation, with proper finally-restore. The MSYS2 test (`:117`) writes `git.exe` under `msys64/ucrt64/bin/git.exe` and `bash.exe` under `msys64/usr/bin/bash.exe`, which is exactly the layout that broke before this fix.
- The `executable()` helper at `:96-101` swallows all stat errors as `false`. Correct for "is this a usable executable" semantics on Windows where stat can fail for permission/lock reasons.

## Risk
Low. The change is additive (PATH is checked first; git-derived fallbacks still run if PATH has no bash); on POSIX `gitbash()` returns early so no behavior change for macOS/Linux users.

## Verdict
**merge-as-is** — clear bug fix, three solid Windows-specific tests, no behavior change off-Windows, careful handling of the WSL system32 trap.
