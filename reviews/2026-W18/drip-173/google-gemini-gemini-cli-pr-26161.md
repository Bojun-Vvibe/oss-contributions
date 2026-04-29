# google-gemini/gemini-cli #26161 — wire shell executable/args from settings

- **PR:** https://github.com/google-gemini/gemini-cli/pull/26161
- **Title:** `fix: shell configuration executable and args not taking effect (especially on Windows)`
- **Author:** cheezmil
- **Head SHA:** 5555a06a539bb8ac84e5b08df6f4800d8410c406
- **Files changed:** 5 (`packages/cli/src/config/config.ts`,
  `packages/cli/src/config/settingsSchema.ts`,
  `packages/core/src/config/config.ts`,
  `packages/core/src/services/shellExecutionService.ts`,
  `packages/core/src/utils/shell-utils.ts`), +65 / −2
- **Verdict:** `merge-after-nits`

## What it does

End-to-end plumbing for two settings keys (`tools.shell.executable` and
`tools.shell.args`) that already had a partial schema entry but were
never read past `loadCliConfig`. Result: users on Windows configuring
`pwsh` via settings were silently getting whatever `getShellConfiguration()`
defaulted to instead.

The change adds five hops:

1. `settingsSchema.ts:1640-1660` — declares both settings (`requiresRestart: true`,
   `showInDialog: true`, with the description explicitly listing
   `pwsh`/`bash`/`zsh` examples).
2. `cli/.../config.ts:1071-1072` — passes `settings.tools?.shell?.executable`
   and `.args` into `loadCliConfig` output.
3. `core/config.ts:723-724, 879-880, 1255-1256, 3479-3486` — stores them
   on `Config`, exposes `getShellExecutable()` / `getShellArgs()`, and
   forwards into `shellExecutionConfig`.
4. `shellExecutionService.ts:106-107, 412-415` — adds the fields to
   `ShellExecutionConfig` and passes them into `getShellConfiguration()`.
5. `shell-utils.ts:649-668` — `getShellConfiguration` now accepts
   `(overrideExecutable?, overrideArgs?)` and short-circuits the OS
   detection block when an override is present.

## What's good

- The override branch in `shell-utils.ts:651-666` is the only meaningful
  logic change; everything else is straight wiring. The wiring matches
  the established pattern (see how `useRipgrep` flows through the same
  five layers) so future readers don't get surprised.
- `argsPrefix` defaulting:

  ```ts
  argsPrefix:
    overrideArgs || (isPowerShell ? ['-NoProfile', '-Command'] : ['-c']),
  ```

  This is the right ergonomics — users who set just
  `tools.shell.executable: pwsh` get the sensible PowerShell prefix
  for free, while `tools.shell.executable: bash` gets `-c`. Matches
  the prior behavior of the OS-detection branch.
- PowerShell detection covers all four common spellings
  (`powershell.exe`, `pwsh.exe`, `powershell`, `pwsh`) via
  `executable.toLowerCase()` then `endsWith` / equality. Reasonable
  for `tools.shell.executable: "C:\\Program Files\\PowerShell\\7\\pwsh.exe"`.
- `requiresRestart: true` on both schema entries is correct — the
  config is captured into `Config` at construction time and never
  re-read.

## Nits / risks

- `executable.toLowerCase().endsWith('pwsh.exe')` on Windows works,
  but on macOS/Linux a path like `/opt/ps/pwsh-preview/pwsh`
  will **not** be detected as PowerShell because it doesn't end in
  `pwsh` (it ends in `pwsh-preview/pwsh`... wait, actually it does
  end in `/pwsh` and `executable === 'pwsh'` requires equality, not
  endsWith). So the literal binary at
  `/opt/ps/powershell/7/pwsh` fails detection because it's
  the full path, not `pwsh`. Fix:

  ```ts
  const base = path.basename(executable.toLowerCase());
  const isPowerShell = base === 'powershell' || base === 'pwsh' ||
                        base === 'powershell.exe' || base === 'pwsh.exe';
  ```

  This makes the detection path-agnostic and removes the four
  almost-duplicate branches.
- `overrideArgs` of `[]` (the schema default) is **truthy** in JS
  (`[] || x` → `[]`), so a user who sets
  `tools.shell.args: []` explicitly will get an empty argsPrefix even
  for PowerShell — meaning `pwsh "echo hi"` instead of
  `pwsh -NoProfile -Command "echo hi"`. Currently the schema
  `default: [] as string[]` may make this fire unintentionally. Use
  `overrideArgs?.length ? overrideArgs : (isPowerShell ? ... : [...])`
  to fall back when the array is empty.
- No tests added. `shell-utils.ts` has a sibling `*.test.ts`; one test
  per branch (override-pwsh, override-bash, override-with-explicit-args,
  override-with-empty-args) would lock in the fix and the nits above.
- The `ShellExecutionConfig` type now has `shellExecutable` and
  `shellArgs` fields (`shellExecutionService.ts:106-107`) but
  callers that build a `ShellExecutionConfig` outside of `Config`
  (tests, plugins) won't be affected unless they opt in. Worth a
  one-line JSDoc on the new fields explaining "ignored unless
  shell-utils.getShellConfiguration is reached via this path".

## Verdict rationale

`merge-after-nits`: the wiring is correct and the feature is genuinely
needed (the schema entries already advertised this capability — they
just didn't work). The two nits — `pwsh` full-path detection and
empty-array truthiness — are real edge cases that will bite Windows
power-users immediately and should be fixed before merge.
