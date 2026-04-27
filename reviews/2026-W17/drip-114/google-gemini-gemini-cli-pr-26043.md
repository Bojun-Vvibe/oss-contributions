# google-gemini/gemini-cli PR #26043 — fix: settings persistence and OAuth URL display (#12137)

- **PR**: https://github.com/google-gemini/gemini-cli/pull/26043
- **HEAD**: `91895425`
- **Touch**: ~30 source files for the actual fix, BUT the diff also contains
  ~50 binary/large blobs under `node20/node-v20.19.0-win-x64/` (Node.js
  Windows distribution: `node.exe`, `npm.cmd`, npm/npx wrappers,
  CHANGELOG.md, LICENSE, etc.) plus a `empty_settings.json` at repo root.

## Context

The intent (per PR title and #12137) is two-fold:

1. Fix a settings-persistence bug where `LoadedSettings.setValue()` would
   `saveSettings(settingsFile)` on every call, defeating any caller that
   wanted to apply multiple updates and then explicitly persist once
   (and forcing migrations to either repeatedly disk-write or to bypass
   the public API).
2. Fix OAuth URL display so the printed URL is usable from the terminal.

## Diff (the intentional half)

`packages/cli/src/config/settings.ts:444-475` adds an `options?:
{ skipSave?: boolean }` parameter to `setValue`, gates the
`saveSettings(settingsFile)` call on `!options?.skipSave`, and exposes a
new explicit `save(scope: LoadableSettingScope)` method that calls
`saveSettings` only if `isPersistable(settingsFile)`. The migration
caller at `:948` is updated to collect `modifiedScopes` from
`migrateDeprecatedSettings(loadedSettings)` and `save` each one
exactly once at the end. That's the right shape — one disk write per
batch instead of one per setting.

The `→ → ￫` ambiguous-width replacements bundled in (`acpResume.test.ts`,
`extensions/update.test.ts:128/170`, `policy-engine.integration.test.ts:556/713`,
several `__snapshots__/*.snap`, and dozens more) appear to be PR #26041
overlapping into this branch — clearly a rebase issue.

## Diff (the *unintentional* half — blocking)

`empty_settings.json` is a 0-byte file at repo root. Almost certainly
a debugging artifact that escaped the commit.

`node20/node-v20.19.0-win-x64/` contains the unpacked Node 20.19.0
Windows binary distribution: `node.exe` (a multi-MB Windows binary),
`npm`, `npx`, `corepack`, `install_tools.bat`, `nodevars.bat`, the full
`CHANGELOG.md` (1706 lines), `LICENSE` (2170 lines), `README.md`,
plus `node_modules/npm/` etc. This is the kind of accidental
`git add .` after extracting a release tarball into a working tree.
Committing this would (a) bloat the repo by tens of MB, (b) mix a
third-party binary distribution into a package licensed differently
from the repo, and (c) ship a Windows `node.exe` that nobody asked for
through the gemini-cli release pipeline.

## Risks / nits

- The settings fix itself is clean and the `save()` carve-out is
  the right API shape. If the author re-rolls without the
  `node20/`, `empty_settings.json`, and the unrelated
  ambiguous-width substitutions, this would be a clean
  merge-after-nits.
- Verify whether `setValue(scope, key, value)` (the old 3-arg
  signature) is called anywhere with a positional `options`-shaped
  fourth arg; the new optional 4th param is back-compat only if every
  call site is using ≤3 args today.
- The `migrateDeprecatedSettings(loadedSettings)` return-shape change
  from `void` to `Set<LoadableSettingScope>` (or array) is API-
  observable; downstream re-implementers should be flagged.

## Verdict

Verdict: request-changes
