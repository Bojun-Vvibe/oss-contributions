# google-gemini/gemini-cli PR #26009 — feat(core): experimental.windowsBash — route shell commands through bash on Windows

- Link: https://github.com/google-gemini/gemini-cli/pull/26009
- Head SHA: `d723ecd5`
- Size: +599 / -14 across ~10 files

## Summary

Adds an opt-in `experimental.windowsBash` boolean setting that, on Windows, routes `run_shell_command` through a `bash -c "<cmd>"` binary discovered via PATH → registry probe (`HKLM\SOFTWARE\GitForWindows`) → well-known install-dir filesystem probes, instead of the hardcoded PowerShell. Rationale in the PR body is structural: foundation models emit Unix shell syntax (`&&`, `grep`, `awk`, `/dev/null`, `find -name`) regardless of system-prompt instructions because the training prior dominates a few lines of context, so the fix meets the model where its training is rather than fighting it. Targets the long Windows-compat tail tracked in #21340.

## Specific-line citations

- `settingsSchema.ts:2287-2302` declares the setting with `default: false` (correct — opt-in keeps default behavior unchanged), `category: 'Experimental'`, `requiresRestart: false`, and a description that names Git-for-Windows as the recommended provider but explicitly accepts WSL/MSYS2/Cygwin and documents the no-bash-found PowerShell fallback.
- `shell-utils.ts:684-757` adds `resolveBashOnPath()` with a module-level `cachedBashPath` cache, `resolveGitForWindowsBash()` doing registry-first then `ProgramFiles` / `ProgramFiles(x86)` / `LOCALAPPDATA\Programs\Git` filesystem probes — three locations cover the "installed for all users", "32-bit installer on 64-bit Windows", and "per-user install" cases. Registry probe has a 3-second `timeout` and is `try`-wrapped to fall through gracefully.
- `shellExecutionService.ts:411-436` is the routing site: bash routing only fires when `enableWindowsBash && isWindows && !isStrictSandbox` — sandbox path correctly takes precedence. Missing-bash warning is gated by a static `windowsBashNotFoundWarned` flag so the noisy warning fires once per session, not per command.
- `shellExecutionService.ts:482-486` sets `MSYS_NO_PATHCONV=1` plus `LANG=LC_ALL=C.UTF-8` when running bash on Windows — correct, prevents MSYS path-mangling that would otherwise rewrite `/c/foo` ↔ `C:\foo` in unpredictable ways and avoids charset surprises with non-ASCII output.
- `integration-tests/windows-bash.test.ts:1-276` is a new integration suite that uses `it.skipIf(!bashPath)` so it auto-runs on Linux/macOS (where bash is always present) and gates on Windows, exercising real bash via `ShellExecutionService.execute` rather than mocks — the right shape for a routing change.

## Verdict

**merge-after-nits**

## Rationale

Approach is correct and the routing is well-isolated. The `enableWindowsBash` plumbs through `ConfigParameters` → `Config` field → `getEnableWindowsBash()` → `getShellDefinition` → `ShellExecutionConfig.enableWindowsBash` (`config.ts:1298`, `coreTools.ts:251-254`, `shellExecutionService.ts:108`) which is the right depth — no global mutable state, no env-var override surface that would let the setting be silently disagreed with. The `isPosixShellEffective(config)` helper at `shell-utils.ts:760` is also a smart addition because downstream tools that need to know "is the effective shell POSIX-ish" can now ask once instead of duplicating the platform+setting check.

Nits worth addressing before merge:

1. **Cache is process-lifetime.** `cachedBashPath` (`shell-utils.ts:684`) is captured on first call and held forever — if a user installs Git-for-Windows mid-session, the cache stays `null` and the warning keeps firing. `clearBashPathCache()` exists but isn't wired to anything user-facing (e.g. `/clear`, settings change). Either add a settings-change hook to clear it, or accept it explicitly in the description ("requires restart after installing bash").
2. **Registry probe assumes 64-bit hive.** The query is `HKLM\SOFTWARE\GitForWindows` — on 32-bit Node running on 64-bit Windows, WoW6432Node redirection means this could miss. Filesystem probes cover most of this gap, but a `/reg:64` flag on the `reg query` would be belt-and-suspenders.
3. **`MSYS_NO_PATHCONV=1` is correct, but `MSYS2_ARG_CONV_EXCL=*` is the canonical wider knob** for users who run MSYS2 bash (not Git-for-Windows). Worth adding alongside since the description explicitly accepts MSYS2.
4. The warn-on-not-found goes to `process.stderr.write` directly rather than through the project's logging surface — fine for "very-early-config-time" but inconsistent with the `debugLogger.warn` call right below it.

None of these are merge-blockers; the feature is opt-in, well-tested (integration suite is the right shape), and the design correctly recognizes that fighting the training prior is a losing strategy.
