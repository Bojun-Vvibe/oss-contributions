# google-gemini/gemini-cli PR #26130 — fix(cli): pass node arguments via NODE_OPTIONS during relaunch to support SEA

- **PR**: https://github.com/google-gemini/gemini-cli/pull/26130
- **Author**: cocosheng-g
- **Merged**: 2026-04-28T21:30:08Z
- **Head SHA**: `376031b2b806`
- **Size**: +389/-126 across 6 files
- **Verdict**: `merge-after-nits`

## Context

The CLI's "lightweight parent process" pattern (in `packages/cli/index.ts`)
spawns a child Node process with extra `--max-old-space-size=...` flags
computed from the user's available RAM (the "memory args" in
`getMemoryNodeArgs`). Pre-PR, the parent built a `nodeArgs` array of
`[...process.execArgv, ...memoryArgs, scriptPath, ...scriptArgs]` and
passed it to `spawn(process.execPath, nodeArgs, ...)`. That works in
standard `node script.js` mode but breaks in SEA (Single Executable
Application) mode where `process.execPath` *is* the binary that contains
the bundled JS — there's no separate "node interpreter" arg, no
`script.js` to point at, and node-flag-style args get forwarded to the
application instead of the interpreter. SEA users hit two failure modes:
(a) memory flags became user-visible CLI args and broke flag parsing,
and (b) on relaunch the binary kept pushing its own path onto its argv,
producing argument-duplication that compounded across relaunches.

## What changed

1. **New helper module** `packages/cli/src/utils/processUtils.ts:31-127`
   exporting four primitives:
   - `isStandardSea()` — true iff in SEA *initial* launch (`IS_BINARY=true`
     OR `process.isSea?.() === true`) AND `argv[0] !== argv[1]`. The
     second clause is the discriminator between an *initial* SEA launch
     (`argv = [/bin/gemini, ...args]`) and a *relaunched* SEA child
     (`argv = [/bin/gemini, /bin/gemini, ...args]`, because the relaunch
     code injects `execPath` as a placeholder script slot — see
     `getSpawnConfig` below).
   - `getScriptArgs()` — returns `process.argv.slice(isStandardSea() ? 1 : 2)`,
     i.e. the user-facing CLI args, accounting for both SEA modes.
   - `isSeaEnvironment()` — broader test: any of `IS_BINARY=true`,
     `process.isSea?.()`, or `argv[0] === argv[1]`. Used to gate the
     spawn-mode branch.
   - `getSpawnConfig(nodeArgs, scriptArgs)` — returns
     `{ spawnArgs, env }` correctly for both modes:
       - **SEA branch** (`processUtils.ts:90-117`): node flags are
         packed into `NODE_OPTIONS` (with the existing
         `process.env['NODE_OPTIONS']` prepended so user overrides are
         preserved), and `spawnArgs = [process.execPath, ...scriptArgs]`.
         The `process.execPath` placeholder is *deliberate* — it
         occupies the `argv[1]` slot so the child's `getScriptArgs()`
         (which uses `slice(2)` once it sees `argv[0] === argv[1]`)
         lines up correctly. Without this placeholder, `argv[1]` would
         be the first user arg and the slice would chop it.
       - **Standard branch** (`processUtils.ts:118-126`): unchanged
         shape `[...process.execArgv, ...nodeArgs, process.argv[1],
         ...scriptArgs]`.

2. **Hardening: reject complex node-arg escaping in SEA mode**
   (`processUtils.ts:97-105`):

   ```ts
   for (const arg of nodeArgs) {
     if (/[\s"'\\]/.test(arg)) {
       throw new Error(
         `Unsupported node argument for SEA relaunch: ${arg}. Complex escaping is not supported.`,
       );
     }
   }
   ```

   This is the right call: `NODE_OPTIONS` is parsed with shell-like
   escaping rules and round-tripping `--title "My App"` through the
   env var requires per-shell quoting that `process.env['NODE_OPTIONS']`
   doesn't give you. Failing fast with a clear error beats silent
   corruption. The codebase only uses `--max-old-space-size=N` style
   flags so this never fires in practice, but the guard is correct.

3. **Call-site simplification** in `packages/cli/index.ts:79-95`:
   the inline `nodeArgs.push(...)` chain is replaced with
   `const { spawnArgs, env: newEnv } = getSpawnConfig(memoryArgs, scriptArgs);`
   and the constants `RELAUNCH_EXIT_CODE`/`GEMINI_CLI_NO_RELAUNCH`
   move into `processUtils.ts`. The `relaunch.test.ts` setup at
   lines 117-130 also gets a tidy: `originalEnv = { ...process.env }`
   is replaced with `vi.stubEnv(...)` calls for the three env vars
   the tests need to control (`GEMINI_CLI_NO_RELAUNCH`, `IS_BINARY`,
   `NODE_OPTIONS`).

4. **Test coverage** (`processUtils.test.ts:44-217`): a 173-line new
   `describe('SEA handling utilities')` block that pins each helper
   independently:
   - `isStandardSea` — 4 cases including the discriminator
     `argv[0] === argv[1]` returning false even when `IS_BINARY=true`.
   - `getScriptArgs` — slice-from-1 for initial SEA, slice-from-2 for
     relaunched-SEA + standard node.
   - `isSeaEnvironment` — 4 cases.
   - `getSpawnConfig` — three cases: standard node, SEA with new
     nodeArgs (asserts `NODE_OPTIONS` is `--existing-flag --max-old-space-size=8192`,
     i.e. the existing env value is *prepended*), and SEA with
     complex escaping (asserts the `Unsupported node argument` throw).

## Why `merge-after-nits`

The structural fix is correct and the test surface is genuinely thorough
for what's a tricky cross-mode launcher. Concerns worth resolving in a
follow-up:

- **`isStandardSea` discriminator is fragile when `argv[0]` is symlinked.**
  The check `process.argv[0] !== process.argv[1]` correctly distinguishes
  initial vs relaunched SEA when `getSpawnConfig` injects exactly
  `process.execPath` as `spawnArgs[0]`. But on a system where the user
  invokes the binary via a symlink (`/usr/local/bin/gemini → /opt/gemini/bin/gemini`),
  `process.execPath` resolves to the realpath but `process.argv[0]` may
  preserve the symlink path (Node 18+ default), so the comparison can
  spuriously read as "not equal" inside a relaunched child. Worth a
  test case: stub `process.argv[0]` to a different string from
  `process.execPath` after a relaunch and verify the slice is still
  correct.

- **`NODE_OPTIONS` concatenation order is `existing + new` — confirm
  the precedence is intentional.** At `processUtils.ts:108`,
  `${existingNodeOptions} ${nodeArgs.join(' ')}.trim()` puts the new
  flags *after* the user-supplied ones. Node's documented behavior for
  `--max-old-space-size` with multiple values is "last wins," so this
  ordering correctly lets the CLI's computed memory bound override a
  user's manual setting. If the *opposite* was intended (user wins
  over CLI), the order should flip. Worth a one-line comment either
  way: "memory bound computed from system RAM intentionally overrides
  user `NODE_OPTIONS`" makes the precedence explicit and survives
  the next refactor.

- **The placeholder-execPath trick is load-bearing and undocumented.**
  At `processUtils.ts:113-117`, `finalSpawnArgs.push(process.execPath,
  ...scriptArgs)` — the `process.execPath` is *not* a redundant duplicate
  of the binary path; it's a deliberate slot-filler so the child's
  `argv[1]` matches `argv[0]` and `getScriptArgs` correctly slices from
  index 2. The 4-line comment block above the push gestures at this
  ("To maintain the [node, script, ...args] structure expected by the
  application (which uses slice(2))") but it would be much clearer to
  also reference the `argv[0] === argv[1]` discriminator in
  `isStandardSea`/`isSeaEnvironment` so the next reader knows the two
  ends of the protocol live together.

- **`process.execPath` lookup happens once per spawn**, but
  `process.execPath` *can* change at runtime (rare, but
  `node --use-openssl-ca` and certain fs reconfigurations affect it).
  Not worth fixing — just note it would be visible if a future
  feature ever toggled it mid-run.

## What I learned

The two-mode launcher pattern (lightweight parent → memory-tuned child)
predates SEA, and SEA breaks the implicit `[execPath, scriptPath,
...args]` argv contract that `slice(2)` was relying on. The clean shape
for fixing it is exactly what this PR does: extract the
"how do I distinguish the two modes" logic into a single helper module
with one named primitive per question (`isStandardSea`, `isSeaEnvironment`,
`getScriptArgs`, `getSpawnConfig`), and pin each primitive with a tight
test rather than letting the launcher carry the conditional logic
inline. The other lesson is the one-shot guard against
`NODE_OPTIONS`-shell-escaping pain: rather than trying to round-trip
arbitrary node-args through an env var with shell semantics, fail fast
on any arg containing whitespace/quotes/backslashes and document that
SEA mode only supports simple `--flag=value` shapes. The codebase only
ever passes those, so the guard is purely defensive — and that's
exactly when guards are most valuable.
