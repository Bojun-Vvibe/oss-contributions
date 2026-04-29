# google-gemini/gemini-cli#26161 — fix: shell configuration executable and args not taking effect (especially on Windows)

- URL: https://github.com/google-gemini/gemini-cli/pull/26161
- Head SHA: `5555a06a539bb8ac84e5b08df6f4800d8410c406`
- Size: +65 / −2, 5 files
- Verdict: **merge-after-nits**

## Summary

Fixes a long-standing bug where `tools.shell.executable` and `tools.shell.args` in `settings.json` were silently ignored because `getShellConfiguration()` had no parameter to receive them. Wires settings → Config (`packages/cli/src/config/config.ts:1068-1072` adds the two fields to `loadCliConfig`'s mapping; `packages/core/src/config/config.ts:1252-1253, 1276-1277, 3477-3485` adds storage, `ShellExecutionConfig` plumbing, and getters), then `getShellConfiguration(overrideExecutable, overrideArgs)` at `packages/core/src/utils/shell-utils.ts:646-665` short-circuits to the user-supplied values when present, defaulting PowerShell args to `["-NoProfile", "-Command"]` and bash-shaped args to `["-c"]`.

## Specific issues flagged

### 1. PowerShell detection by suffix-match misses common installation paths

At `shell-utils.ts:653-657`:

```ts
const executable = overrideExecutable.toLowerCase();
const isPowerShell =
  executable.endsWith('powershell.exe') ||
  executable.endsWith('pwsh.exe') ||
  executable === 'powershell' ||
  executable === 'pwsh';
```

Misses these legitimate paths:

- `C:\Program Files\PowerShell\7\pwsh` (no `.exe` suffix on the final segment but a full path → fails both `endsWith` and `===`);
- `pwsh-preview`, `pwsh-lts` aliases that some MS-installer flavors register;
- POSIX-style `/usr/local/bin/pwsh` on macOS/Linux for cross-platform PowerShell users (passes the `endsWith('pwsh')` check by coincidence — wait, no, `endsWith('pwsh.exe')` is false and `=== 'pwsh'` is false because `executable` is the full path).

Switch to: `const base = path.basename(executable, path.extname(executable)).toLowerCase(); const isPowerShell = base === 'powershell' || base === 'pwsh' || base.startsWith('pwsh-');`. Also handles the suffix variants without listing them all.

### 2. `requiresRestart: true` on `executable` and `args` is correct but should be tested

At `packages/cli/src/config/settingsSchema.ts:1640-1660` both new fields declare `requiresRestart: true`. Good — `ShellExecutionService` reads the resolved values at construction time. But there's no test asserting that mid-session `settings.json` edits to `tools.shell.executable` are ignored until restart (a user might reasonably expect hot-reload). Either add a test that pins "no hot-reload" or document it inline in the description text at `:1644`.

### 3. `argsPrefix` fallback logic differs subtly from `getShellConfiguration`'s POSIX defaults

The override branch at `shell-utils.ts:660-664` returns:

```ts
return {
  executable: overrideExecutable,
  argsPrefix: overrideArgs || (isPowerShell ? ['-NoProfile', '-Command'] : ['-c']),
  shell: isPowerShell ? 'powershell' : 'bash',
};
```

The non-override branch (the existing `getShellConfiguration()` body, untouched) likely returns different `argsPrefix` for `cmd.exe` (which gets `['/c']`). When a user sets `tools.shell.executable: "cmd.exe"` explicitly (a legitimate workflow on Windows), they fall into the override branch and get `argsPrefix: ['-c']` — which `cmd.exe` does not understand and will silently treat as a literal first argument. Add a third detection arm: `const isCmd = base === 'cmd';` → `argsPrefix: ['/c']`, `shell: 'cmd'`.

### 4. `overrideArgs || (default)` truthy-check breaks empty-array intent

At `:662`: `argsPrefix: overrideArgs || (isPowerShell ? ['-NoProfile', '-Command'] : ['-c'])`. If a user explicitly sets `tools.shell.args: []` to mean "no extra args, just exec the shell directly", `[]` is truthy in JS so this works *correctly* (passes the `[]`). But if `overrideArgs` is `undefined` (the schema default per `settingsSchema.ts:1657` is `[] as string[]`)... wait, the schema default is `[]`, so `overrideArgs` is always an array, never `undefined`. That means the `|| (default)` fallback **never fires** and PowerShell users who don't set `args` get `[]` instead of the documented `-NoProfile -Command` defaults. Either:

- Change schema default at `:1657` from `[] as string[]` to `undefined as string[] | undefined`, OR
- Check `Array.isArray(overrideArgs) && overrideArgs.length > 0` instead of truthy.

This is the kind of bug that exactly matches the title of this PR ("...not taking effect") — so worth a unit test for the empty-array case.

### 5. `shell: 'bash'` for non-PowerShell executable is incorrect for zsh/fish/etc.

At `:663`: `shell: isPowerShell ? 'powershell' : 'bash'`. If `shell` is consumed downstream for syntax-aware quoting / heredoc / backslash handling, calling zsh `'bash'` may produce subtly wrong escaping. Check whether `ShellConfiguration.shell` is informational-only (just a label) or load-bearing for command-construction. If load-bearing, add `'zsh' | 'fish' | 'sh'` cases or a generic `'posix'` umbrella.

## Why merge-after-nits

Real fix to documented-but-broken config surface. Items 1, 3, 4 are correctness on Windows / cross-platform paths that this PR explicitly claims to fix in the title — worth resolving before merge for an honest title. Item 2 is testing, item 5 is downstream-correctness depending on consumer.
