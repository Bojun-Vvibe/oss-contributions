# google-gemini/gemini-cli#26261 — Skip binary CLI relaunch

- PR: https://github.com/google-gemini/gemini-cli/pull/26261
- Head SHA: `2d5f9d0fcd47a99b46d44e1691c642a85ef23a99`
- Author: ruomengz
- Files: 4 changed (`packages/cli/index.ts`, `packages/cli/src/gemini.tsx`,
  `scripts/build_binary.js`, new
  `.github/workflows/build-unsigned-mac-binaries.yml`), +71 / −2

## Context

The Gemini CLI uses a parent/daemon launch model: `packages/cli/index.ts`
runs as a lightweight parent that, after determining the right Node memory
arguments, re-execs itself with `--max-old-space-size=...` (and friends)
into a heavier child process that does the real work. This relaunch saves
~1.5s of cold-start by deferring heavy imports out of the parent.

That model breaks when the CLI is shipped as a Single Executable
Application (SEA) — Node SEA bundles cannot accept the same `node`
arguments at exec time, and the relaunch step either fails outright or
reinvokes the SEA binary in a way that doesn't apply the requested heap
limit.

## Design

The PR adds an `IS_BINARY=true` env-var sentinel and short-circuits both
the relaunch site and the memory-args computation when set.

**Relaunch gate (`packages/cli/index.ts:78-82`).**
```ts
if (
  !process.env['GEMINI_CLI_NO_RELAUNCH'] &&
  !process.env['SANDBOX'] &&
  process.env['IS_BINARY'] !== 'true'
) {
  // --- Lightweight Parent Process / Daemon ---
  ...
}
```
Adds the `IS_BINARY !== 'true'` clause to the existing two-clause guard.
Pre-PR, `GEMINI_CLI_NO_RELAUNCH` and `SANDBOX` were the only escape
hatches; the SEA case now joins them.

**Memory-args gate (`packages/cli/src/gemini.tsx:128-134`).**
```ts
if (
  process.env['IS_BINARY'] === 'true' ||
  process.env['GEMINI_CLI_NO_RELAUNCH'] ||
  process.env['SANDBOX']
) {
  return [];
}
```
Returns an empty memory-args array when in any of the three "don't
relaunch" modes. Pre-PR, only `GEMINI_CLI_NO_RELAUNCH` skipped the
memory-args branch; this PR adds the `IS_BINARY` and `SANDBOX` cases.
Importantly, this **also adds `SANDBOX`** to the memory-args skip list,
which is a behavior change beyond what the title implies — pre-PR, a
sandboxed launch returned to the relaunch path's prior step (skipped in
`index.ts`) but `getNodeMemoryArgs` would still build memory args that
nothing would consume. New code returns `[]` in all three skip cases,
keeping `index.ts` and `gemini.tsx` consistent. Worth calling out in the
PR description.

**Build script (`scripts/build_binary.js:101-105`).**
```js
if (process.env.SKIP_SIGNING === 'true') {
  console.log(`Skipping signing for ${filePath} (SKIP_SIGNING=true)`);
  return;
}
```
Adds an opt-out for code signing during binary builds, gated on the
`SKIP_SIGNING` env var. Required for the new unsigned-binaries workflow
below to actually succeed without an Apple Developer ID in CI.

**Workflow (`.github/workflows/build-unsigned-mac-binaries.yml`, +56 LOC).**
A new manually-triggered (`workflow_dispatch`) workflow that builds
unsigned x64+arm64 Mac SEA binaries via `npm run build:binary` with
`SKIP_SIGNING=true`, verifies the output exists at
`dist/darwin-${arch}/gemini`, and uploads as a 5-day-retention artifact.
Uses ratcheted action SHAs (`actions/checkout@34e114876b0b...`,
`actions/setup-node@49933ea5288c...`,
`actions/upload-artifact@ea165f8d65b6e75b...`) — good supply-chain
hygiene.

## Risk analysis

**`IS_BINARY` env-var precedence.** `process.env['IS_BINARY']` is now
load-bearing for SEA. Worth thinking about what happens if a user
accidentally sets it for a non-SEA `node packages/cli` invocation: they
get the no-relaunch path, the in-process Node loses the heap-limit boost,
and very large contexts will OOM where they previously didn't. Low
likelihood (it's a non-standard env var) but the failure mode is silent
(slow) rather than loud. Two options: (a) auto-detect SEA (`process.argv0`
ends with the binary name + Node has `process.execArgv` empty), or
(b) at minimum log a one-line "running in binary mode, daemon relaunch
skipped" so a debugger can see why a heap-limit isn't applied. (a) is
nicer; (b) is a one-liner.

**`SANDBOX` semantics drift in `gemini.tsx`.** Adding `SANDBOX` to the
memory-args skip list in `getNodeMemoryArgs` is a behavior change for the
sandboxed run path that the PR title and description don't mention.
Looking at how `getNodeMemoryArgs` is called: since `index.ts` already
skips relaunch for `SANDBOX`, the only way `getNodeMemoryArgs`'s old
return value mattered was if some other call site passed it through to
the in-process Node — which there shouldn't be, but it's worth
confirming with `git grep getNodeMemoryArgs`. If the function is only
called by the relaunch site, this change is a no-op for the sandbox case
and removing the `||` clause would be cleaner.

**`SKIP_SIGNING` opt-out.** Skipping code signing is exactly what the
new workflow wants, but the same env var would let a developer
accidentally ship an unsigned production binary if it leaked into a
release pipeline. The signing function is now branchless on
`SKIP_SIGNING=true` — a sturdy defense would be to additionally check
`process.env.GITHUB_WORKFLOW` matches the unsigned-build workflow's name
or refuse `SKIP_SIGNING=true` outside CI. Out of scope, but worth a
follow-up issue.

**Workflow surface.** The new workflow is `workflow_dispatch`-only, no
push/PR triggers, no auto-publish. Artifacts are 5-day-retention. The
runner is `macos-latest`. All good defaults — this is a "manually
triggered, ephemeral, no-secrets" build pipeline, which is exactly the
right shape for an unsigned-binary CI surface.

## Verdict

**`merge-after-nits`**

The core change is small, correct, and the right shape: the SEA case
needed an explicit env-var skip rather than trying to infer SEA-ness from
runtime state. The supporting workflow is a clean addition with proper
SHA-pinned actions.

Three nits before merge:

1. **PR description should mention the `SANDBOX` addition to
   `getNodeMemoryArgs`.** It's a small thing but the title says "Skip
   binary CLI relaunch" and a reviewer expecting only `IS_BINARY` work
   shouldn't have to grep to find the second skip-list entry.
2. **Add a startup log when `IS_BINARY=true` is honored.** One line
   ("Running in binary mode, daemon relaunch skipped") makes the silent
   no-heap-limit failure mode debuggable.
3. **Consider auto-detecting SEA** instead of (or in addition to)
   requiring an env var. `require('node:sea').isSea?.()` (Node ≥ 20.12)
   is the canonical check; falling back to the `IS_BINARY` env var when
   that API isn't available keeps the workflow's explicit set working
   while removing the foot-gun for users who don't know the env var
   exists.

The `SKIP_SIGNING` follow-up (refuse to skip outside CI) is worth a
separate issue but doesn't need to gate this PR.

## What I learned

The "lightweight parent + heavy child" daemon pattern that this CLI uses
to manage Node heap limits is a known headache for SEA builds because SEA
intentionally constrains what `node` arguments can be re-injected. The
right escape hatch is exactly what this PR adds: an env var the binary
sets on itself before re-exec is even contemplated, short-circuiting the
relaunch-and-pass-flags path. The interesting design question is whether
to derive that signal from runtime detection (`sea.isSea()`) or from an
explicit env var — explicit is more robust for now (doesn't depend on a
specific Node version), runtime detection is more user-friendly long-term.
The PR picks explicit, which is the right call for the immediate need.
