# google-gemini/gemini-cli #25977 — feat(extensions): show package.json version alongside config tag

- **Repo**: google-gemini/gemini-cli
- **PR**: #25977
- **Author**: Anjaligarhwal
- **Head SHA**: 79c8251ae195a756434a5c380e980cf0b454d64f
- **Size**: +221 / −6 across eight files; load-bearing edits at
  `packages/cli/src/config/extension.ts` (+38/−0, two new helpers),
  `packages/cli/src/config/extension-manager.ts` (+13/−2),
  `packages/cli/src/config/extensions/update.ts` (+9/−3),
  `packages/cli/src/commands/extensions/update.ts` (+8/−1),
  `packages/core/src/config/config.ts` (+7/−0, struct field).
  Test surface: `extension.test.ts` +106, `extensions/update.test.ts` +33,
  `test-utils/createExtension.ts` +7.
- **Closes**: #22932. **Supersedes**: #23105.

## What it changes

Surfaces the concrete `package.json` version of an extension alongside the
`gemini-extension.json` config version, so extensions installed via a
generic tag (`latest`, `main`) show useful info: `chrome-devtools-mcp
(latest (0.20.2))` instead of `chrome-devtools-mcp (latest)`.

The deliberate architectural move (and the reason this supersedes #23105):
read `package.json` exactly *once* at extension load time and carry the
result on the in-memory extension object, rather than doing
`fs.readFileSync` at every render/use site (which #23105's reviewer bot
flagged as three separate high-priority blocking-I/O regressions).

Concrete edits:

1. **New struct field** `GeminiCLIExtension.packageVersion?: string` at
   `packages/core/src/config/config.ts:392-400` with a doc comment
   explaining "Captured once at load time so display code can show
   meaningful info when the config version is a generic tag like
   `latest`."
2. **Two helpers in `packages/cli/src/config/extension.ts:71-107`:**
   - `getPackageVersion(extensionDir): Promise<string | undefined>` — uses
     `fs.promises.readFile` (async, matches surrounding `_buildExtension`
     style), parses JSON, runtime-checks `typeof pkg.version === 'string'`
     so a non-string `version` (e.g. `123` or missing) doesn't inject
     unexpected types into display code. Returns `undefined` for any
     failure (missing file, parse error, non-string version).
   - `formatVersion(configVersion, packageVersion?)` — pure, no I/O.
     Returns `${configVersion} (${packageVersion})` only when the two
     differ; otherwise just `configVersion`. Treats empty-string
     packageVersion as missing.
3. **Single load-time embed** at `extension-manager.ts:962-967`:
   `_buildExtension` (the canonical async load path called for both
   initial load and post-install reload) `await getPackageVersion(...)`s
   exactly once and stores the result on the extension object.
4. **Three display-site edits** consume the embedded `packageVersion`:
   - `commands/extensions/update.ts:58-67`: replaces
     `${extension.name} (${extension.version})` with the formatVersion
     call.
   - `config/extension-manager.ts:1118-1124` (`toOutputString`): same
     replacement pattern.
   - `config/extensions/update.ts:97-128`: `originalVersion` and
     `updatedVersion` are now both `formatVersion`-wrapped (was two
     separate sync `fs.readFileSync` reads per update pre-PR via
     #23105).
5. **Test infrastructure** at `test-utils/createExtension.ts:30,60-65`:
   adds an optional `packageJson` parameter so tests can opt into
   creating an extension with a `package.json` without boilerplate.

## Strengths

- **Right architectural choice** — the "read once at load, carry on
  object, format at display" pattern is unambiguously correct over
  "read at every render". The PR body explicitly cites the bot
  feedback on #23105 ("the right intent but performed synchronous
  I/O at every render/use") and applies the lesson cleanly. This is
  textbook supersession.
- **`getPackageVersion` is rigorously defensive.** The triple guard —
  missing file → caught (try/catch returns undefined), parse error →
  caught (same), non-string `version` field → typeof guard at
  `:88-89` returns undefined — covers all three failure modes a
  hostile or malformed `package.json` could surface. The four pinned
  cases at `extension.test.ts:2424-2440`
  (`returns the version from package.json when present`,
  `returns undefined when package.json does not exist`,
  `returns undefined for malformed JSON`,
  `returns undefined when version is missing or non-string`) hit each
  branch.
- **`formatVersion` semantics are exactly right.** Three pinned cases
  at `:2398-2415` cover (1) no packageVersion → return configVersion
  unchanged, (2) packageVersion equals configVersion → no
  redundant `(0.20.2 (0.20.2))`, (3) different → append in parens.
  The empty-string-as-missing case at `:2412-2414` is the right edge:
  a `package.json` with `"version": ""` is treated like a missing
  version rather than producing `latest ()`.
- **The struct field is optional** (`packageVersion?: string`), so
  this is a strictly additive change for downstream consumers of
  `GeminiCLIExtension`. Extensions loaded by older code paths (or
  ones without a `package.json`) keep working with `packageVersion`
  undefined and the old display behavior is preserved verbatim by
  `formatVersion`'s no-packageVersion branch.
- **Integration tests pin both load-time and display-time contracts**
  at `extension.test.ts:254-294`:
  - `should populate packageVersion from package.json when present` —
    pins the `_buildExtension` embed.
  - `should leave packageVersion undefined when package.json is
    absent` — pins the no-package.json fallback.
  - `toOutputString includes package.json version when it differs
    from the config version` — pins the display-side format directly
    via `extensionManager.toOutputString(extensions[0])`.
  Plus the `extensions/update.test.ts:204-228` end-to-end test
  asserts `originalVersion: 'latest (0.20.2)'` /
  `updatedVersion: 'latest (0.20.4)'` at the `updateExtension`
  return shape, locking the update flow's display contract.
- **Tests use `mkdtempSync` + `rm` cleanup** at `:2419-2427` rather
  than mocking `fs`, which gives genuine integration-level coverage
  of the read path. The malformed JSON case literally writes
  `'{not valid'` to a tmp file and asserts the parse failure is
  caught. That's the right level of fidelity for a file-I/O helper.
- **`createExtension` test util** at `:30,60-65` is correctly
  optional-additive — the existing call sites that don't supply
  `packageJson` produce extensions without a `package.json`
  unchanged.

## Concerns / nits

- **`getPackageVersion` opens the file even when a `package.json`
  doesn't exist**, paying the syscall cost via
  `fs.promises.readFile`'s ENOENT path. For users with hundreds of
  extensions installed (rare but possible — e.g. company-internal
  extension marketplaces), that's hundreds of failed-open syscalls
  during initial load. A pre-check via `fs.promises.access(pkgPath,
  fs.constants.R_OK)` would still pay the same cost; the cleaner
  optimization would be to make the try/catch path log at debug-only
  level so the failure mode is observable, and to consider whether
  `fs.promises.stat` (which can short-circuit on ENOENT slightly
  faster than `readFile`) is worthwhile. Probably premature
  optimization — leave as-is.
- **The eslint-disable at `:84` (`@typescript-eslint/no-unsafe-type-assertion`)**
  is correct for the `as { version?: unknown }` cast — the JSON parse
  result is genuinely `unknown` and the cast is the minimum-surface
  way to introspect the shape. Worth a comment explaining "we cast
  to a narrow shape immediately and runtime-check `typeof
  pkg.version === 'string'` before use" so a future cleanup PR
  doesn't try to remove the type assertion and silently break the
  defensive guard. Currently the inline comment says nothing about
  the runtime check.
- **`formatVersion` is exposed publicly via the `extension.ts` module
  re-export** (the test mock at `extensions/update.test.ts:43-48`
  shadows it). That's fine for now, but as a public API surface it
  inherits the contract that "two equal version strings collapse" —
  if a future caller actually wants both versions side-by-side
  (e.g. for diagnostic output), they'd have to bypass the helper.
  Not a blocker; just worth a one-line docstring noting the
  intentional collapse.
- **The `formatVersion` mock at `extensions/update.test.ts:43-48`**
  reimplements the helper's logic in the test rather than importing
  the real function. That's the standard mocking pattern for tests
  that want to isolate the unit under test, but it does mean a
  future change to `formatVersion`'s collapse logic (e.g. adding
  `(unchanged)` to equal-version output) would silently desync the
  mock from production. A `vi.fn().mockImplementation(formatVersion)`
  pattern that imports the real helper would catch that, at the cost
  of slightly less unit-test isolation.
- **The PR doesn't update the `extensions list` user-facing
  documentation.** The display string `chrome-devtools-mcp (latest
  (0.20.2))` is genuinely a new format users will see; if there's
  user-facing docs (e.g. `docs/cli/extensions.md`) describing the
  list output, they should reflect the new format. Not visible in
  the diff, so worth a quick search before merge.
- **Windows symlink EPERM caveat acknowledged in PR body** is the
  right disclosure — `Validated on Windows ... the only failing
  tests are pre-existing Windows symlink EPERM issues that also
  fail on main and require Developer Mode to pass`. Worth a CI
  note for maintainers so this PR isn't blocked on a known
  preexisting failure.

## Verdict

**merge-as-is.** Architecturally sound supersession of #23105 that
fixes the reviewer-bot's three blocking-I/O complaints by relocating
the read to load-time and carrying the result on the in-memory
extension object. Defensive helper, complete branch coverage on both
unit and integration levels, additive struct field that doesn't break
downstream consumers, and a test-utility extension that's
optional-additive. The nits above are documentation polish (eslint
comment, public-API docstring, mock-vs-real tradeoff) rather than
correctness blockers — none change the diff.
