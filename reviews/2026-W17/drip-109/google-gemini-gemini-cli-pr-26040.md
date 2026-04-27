# google-gemini/gemini-cli #26040 — fix: auto-detect windows native sandbox and disable input blocker by default

- **Repo**: google-gemini/gemini-cli
- **PR**: #26040
- **Author**: pawanwashudev-official
- **Head SHA**: 3184b993eb0c026d15cdf021dba4c43b40a4491e
- **Size**: +42 / −5 across seven files; load-bearing edits at
  `packages/cli/src/config/sandboxConfig.ts` (+3/−1),
  `packages/core/src/agents/browser/browserManager.ts` (+2/−1),
  `packages/core/src/agents/registry.ts` (+1/−0),
  `packages/cli/src/config/settingsSchema.ts` (+1/−1, default flip),
  `packages/core/src/config/config.ts` (+2/−2, default flip + comment).

## What it changes

Two related but logically distinct fixes bundled into one PR:

1. **Auto-detect Windows native sandbox at
   `packages/cli/src/config/sandboxConfig.ts:104-108`.** Adds a new branch
   to `getSandboxCommand`: after the existing `darwin → sandbox-exec`
   check, before the `commandExists.sync('docker')` check, the new
   `else if (os.platform() === 'win32') return 'windows-native';` makes
   Windows pick the native sandbox by default rather than requiring the
   user to run docker/podman or set `--sandbox-command` explicitly. The
   pinned test at `sandboxConfig.test.ts:157-169` mocks
   `os.platform()` to `'win32'`, mocks every `commandExists.sync` to
   `false` (no docker/podman), and asserts the resulting config is
   `{ enabled: true, command: 'windows-native', ... }`.

2. **Flip `disableUserInput` default from `true` to `false`** at
   `settingsSchema.ts:1320` and the matching predicate flip at
   `config.ts:3618-3623`: `shouldDisableBrowserUserInput()` was
   `browserConfig.customConfig?.disableUserInput !== false &&
   !browserConfig.customConfig?.headless` (i.e. "block input unless
   explicitly opted out"); it's now
   `browserConfig.customConfig?.disableUserInput === true &&
   !browserConfig.customConfig?.headless` (i.e. "allow input unless
   explicitly opted in"). The doc comment at `:374` is updated to
   match: `Default: true in non-headless mode` → `Default: false in
   non-headless mode`.

3. **Mark `windows-native` as a non-container sandbox** at
   `browserManager.ts:587-590` and `registry.ts:277-281`. Both files
   guard `isContainerSandbox` with a new `sandboxType !==
   'windows-native'` clause so the existing container-specific browser
   plumbing (host.docker.internal IP resolution, isolated profiles)
   doesn't fire for the Windows native sandbox case. The new test at
   `browserManager.test.ts:1346-1366` exercises the
   `SANDBOX=windows-native` + `sessionMode: 'existing'` path and asserts
   that `--autoConnect` is preserved, `--isolated` is not added, and
   `--headless` is not forced.

4. **Cosmetic comment update at `browserManager.ts:594`**: "Seatbelt
   sandbox: ... Persistent profiles may conflict with seatbelt
   restrictions" → "Persistent profiles may conflict with sandbox
   restrictions" (s/seatbelt/sandbox/, since the comment now applies to
   multiple sandbox flavors).

## Strengths

- **The auto-detect is the right shape.** Pre-fix, every Windows user
  who didn't manually configure `--sandbox-command` got
  `commandExists.sync('docker')` and `commandExists.sync('podman')`
  evaluated, both probably false on a typical Windows dev box, so they
  fell through to "no sandbox" silently. The new branch hardcodes
  `windows-native` for `win32` regardless of docker/podman presence,
  which matches the `darwin → sandbox-exec` precedent. Test at
  `:157-169` pins this exact path with `commandExists.sync` mocked to
  always-false, so the precedence is locked.
- **The non-container guard is defensive in the right places.** Adding
  `sandboxType !== 'windows-native'` to the `isContainerSandbox`
  predicate at both `browserManager.ts:589` and `registry.ts:280`
  prevents the browser agent from trying to resolve `host.docker.internal`
  (which doesn't exist on Windows native) or applying container-specific
  arg munging. The two-line change is clearly load-bearing.
- **The `disableUserInput` default flip is correct on principle.** A
  CLI tool that *defaults* to blocking user input on a browser window
  during automation is the wrong default for an interactive tool —
  users who want to watch and intervene need it off, users who want
  hands-off automation can enable it. The flip is a one-line schema
  edit + a one-line predicate logic fix, and the doc comment is
  updated to match.
- **The browser-manager test at `:1346-1366`** is the right shape:
  stubs `SANDBOX=windows-native` env var, asserts the StdioClientTransport
  args don't drift toward container-mode, and explicitly asserts
  `--headless` is NOT forced for existing-mode (the comment "Headless
  should NOT be forced for existing mode in windows-native" names
  the contract).
- **Comment cleanup is timely.** The `s/seatbelt/sandbox/` at `:594`
  generalizes the existing safety comment now that multiple sandbox
  types reach that code path.

## Concerns / nits

- **The `disableUserInput` default flip is a behavior change with no
  migration story.** Every existing user with the previous default
  enabled (browser input blocker active during automation) will see
  their browser windows become interactively responsive after upgrade,
  which may surprise automated workflows that relied on the
  block-by-default safety. There's no settings migration that flips
  existing non-default values. Worth either (a) a release-notes entry
  flagging the default change explicitly, (b) a one-shot migration
  prompt that asks users to confirm the new default, or (c) keeping
  the default `true` in `settingsSchema.ts` but fixing only the
  predicate-equivalence bug at `config.ts:3618` (which read `!== false
  && !headless` — that's "default true if undefined" — and could be
  changed to `=== true && !headless` only if the schema default also
  flips, which is what this PR does). The two changes need to be
  evaluated together, and the PR body doesn't acknowledge the
  user-visible behavior shift.
- **No test for Windows `commandExists.sync('sandbox-exec') === true`**
  edge case. The new branch order at `sandboxConfig.ts:104-110` is
  `darwin → sandbox-exec` then `win32 → windows-native` then
  `docker`/`podman`. If a Windows machine somehow has a binary named
  `sandbox-exec` on PATH (unlikely but possible — WSL or a Linux
  emulation layer), the first branch's `os.platform() === 'darwin'`
  guard saves us. Good. But there's no test pinning this. Minor.
- **The `windows-native` sandbox command isn't defined in this PR.**
  The branch returns the literal string `'windows-native'`, but who
  consumes it? If no other call site recognizes `'windows-native'`
  as a real sandbox command, the auto-detect will set the field but
  the subsequent `executeCommand`-style call will fail with "unknown
  sandbox command". A reviewer would want to confirm there's an
  upstream `windows-native` sandbox runner in the codebase (likely
  `packages/core/src/sandbox/windows-native.ts` or similar) and that
  the string `'windows-native'` is actually wired up. The PR's diff
  alone doesn't show this — worth either a cross-reference in the PR
  body or a smoke test that attempts to launch `windows-native`.
- **The `BrowserAgentCustomConfig.disableUserInput` jsdoc** at
  `config.ts:374` is updated to "Default: false in non-headless mode"
  but it doesn't say what happens in headless mode (the predicate at
  `:3621` short-circuits on `!browserConfig.customConfig?.headless`,
  meaning headless mode always returns `false` — i.e. user input
  isn't blocked because there's no visible window to block input on).
  The `browserConfig.customConfig?.disableUserInput === true` clause
  is then irrelevant in headless mode, which is the correct behavior
  but worth noting in the doc as "Default: false; ignored in headless
  mode (no visible window)".
- **The `test_descovery_metadata` style misspelling lives elsewhere
  in the codebase too.** Out of scope for this PR but worth noting:
  the new test name `should preserve --autoConnect when in
  windows-native sandbox with existing mode` correctly avoids
  inheriting any preexisting typos in the surrounding test file.
- **Two genuinely orthogonal fixes in one PR.** The auto-detect-windows
  change and the disable-user-input default flip are independent — a
  user could want one without the other. Splitting into two PRs would
  give each fix its own review focus and allow independent revert/rollback
  if either causes issues. The PR body doesn't motivate the bundling.

## Verdict

**merge-after-nits.** The Windows native sandbox auto-detect is correct
and pinned by a focused test; the non-container guard at
`browserManager.ts`/`registry.ts` is the right defensive addition. The
`disableUserInput` default flip is defensible on principle but is a
user-visible behavior change that needs explicit acknowledgment. Before
merge: (1) verify `'windows-native'` is actually a recognized sandbox
command downstream and add a smoke test if not, (2) split or at least
explicitly motivate the two-fix bundling in the PR body, (3) add a
release-notes entry for the `disableUserInput` default flip, and (4)
extend the jsdoc to clarify headless-mode behavior.
