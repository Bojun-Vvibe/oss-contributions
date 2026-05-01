# QwenLM/qwen-code #3778 — feat(desktop): Add desktop app package with Qwen ACP SDK integration

- **PR**: https://github.com/QwenLM/qwen-code/pull/3778
- **Head SHA**: `179df182a3e2a8ae1dc0c7a550b857b0cbd242df`
- **Files reviewed (representative — full diff exceeded GitHub's 300-file limit; reviewed via metadata + sample)**:
  - `package.json` (+2 -1) — root workspace exclusion of `packages/desktop`
  - `package-lock.json` (+2 -2)
  - `packages/desktop/` (NEW package, ~1495 files, ~324k LOC)
    - `apps/cli/src/index.ts` (+1922) — main CLI entry
    - `apps/cli/src/client.ts` (+239) + `client.test.ts` (+409)
    - `apps/cli/src/server-spawner.ts` (+146)
    - `apps/cli/src/run.test.ts` (+427)
    - `apps/cli/src/commands.test.ts` (+391)
    - `apps/electron/electron-builder.yml` (+187)
    - `apps/electron/build/entitlements.mac.plist` (+14)
    - `Dockerfile.server` (+111)
    - `LICENSE` (+191), `NOTICE`, `SECURITY.md`, `TRADEMARK.md`
    - `.github/ISSUE_TEMPLATE/` (bug + feature templates)
    - `.github/workflows/validate.yml`, `validate-server.yml`
- **Date**: 2026-05-01 (drip-228)

## Context

Adds an entire downstream-fork desktop application
(`packages/desktop/`, described in the PR body as a "Craft/Claude
Desktop fork") into the monorepo, integrated against the Qwen ACP SDK
for skill discovery / session management / context usage. Two
non-package fixes ride along:

1. Workspace-exclusion of `packages/desktop` from the root npm
   workspace (desktop uses bun, monorepo uses npm — keeping them
   distinct prevents lockfile collisions).
2. `DispatchHandler → DispatchHandlers` undici type fix in
   `network-proxy.ts`.

## Observations

1. **Scope-vs-monorepo concern is the load-bearing question.**
   Adding 324k LOC and 1495 files in a single PR is a categorically
   different operation from a normal feature add — it pulls in an
   independently-developed application's full surface area
   (Electron build config, Dockerfile, GitHub workflow templates,
   issue templates, security policy, license, trademark notice) into
   a CLI repo whose tests, security scanning, dependency policy,
   and contributor model were not designed for an Electron app.
   The PR should require the maintainers to explicitly OK absorbing
   that ongoing maintenance burden — desktop UI bugs, Electron
   security patches, npm-vs-bun lockfile sync, two SECURITY.md
   files, two TRADEMARK.md files. Most monorepos handle this via a
   submodule or sibling repo, not in-tree absorption.

2. **Workspace exclusion is the single right detail in
   `package.json`.** Putting `"workspaces": ["packages/*",
   "!packages/desktop"]` (or equivalent) is the load-bearing
   defense against root-`npm install` reaching into the bun-managed
   subtree. This is correctly called out as the "reviewer focus" in
   the PR body. Reviewer should verify (a) the negation pattern
   actually works in the npm version pinned by the repo (older npm
   versions don't honor `!` workspace negation — needs npm ≥ 7), and
   (b) CI does not run a `npm ci` that would still attempt the
   subtree.

3. **Validation is thin for a 324k-line change.** PR body's
   validation section reads:
   > Commands run: `cd packages/desktop && bun install && bun run electron:dev`

   A single dev-mode boot is not adequate validation for a desktop
   app addition. At minimum: (a) electron-builder produces a valid
   artifact on each of macOS/Windows/Linux; (b) the bundled ACP SDK
   integration round-trips a real session/skill/context call;
   (c) the npm-vs-bun lockfile separation actually holds under
   `npm ci` from a clean state. The PR body explicitly admits "Not
   covered: Desktop app Electron build/run on Windows/Linux;
   integration tests for the desktop package" — which is exactly the
   coverage that should exist before in-tree absorption.

4. **License / NOTICE / TRADEMARK additions warrant separate review.**
   `packages/desktop/LICENSE` is 191 lines and `NOTICE`/`TRADEMARK.md`
   are non-trivial. Bringing in an upstream fork's license file
   inside the existing monorepo creates a license-overlay question
   (does the root LICENSE govern, or does the per-package LICENSE
   take precedence, or are both required for distribution?). This is
   typically a maintainer-or-counsel call, not a code-review call.

5. **`DispatchHandler → DispatchHandlers` undici fix is correct.**
   Confirmed via undici typings — `DispatchHandlers` is the
   plural-named interface for the per-method handler shape; the
   singular form `DispatchHandler` is a different (often older) type.
   This is a real bug fix that would be merge-as-is on its own. It
   shouldn't be bundled with the 324k-line desktop import.

6. **Two CI workflows added under `packages/desktop/.github/`** —
   `validate.yml` and `validate-server.yml`. The `.github/`
   subdirectory **inside a package** is generally a no-op (GitHub
   only honors workflows at the repo-root `.github/workflows/`).
   These files will exist but never run as workflows. Either lift to
   root with a `paths:` filter scoped to `packages/desktop/**`, or
   delete them with a follow-up note that desktop CI lives elsewhere.

## Verdict

**needs-discussion** — the substance of the change (desktop app
integration via ACP SDK, workspace separation, undici type fix) is
not the question; the question is whether the maintainer team has
agreed to absorb the ongoing maintenance burden of a 1495-file
Electron fork in-tree, with all the cross-platform CI / Electron
security / dual-license / dual-trademark / dual-SECURITY.md tax that
implies. That is a maintainer-policy call, not a reviewer call. If
the maintainers OK the in-tree model, the PR should be split:
(a) the undici type fix as a standalone PR, (b) the workspace
configuration change as a standalone PR with a CI assertion, then
(c) the desktop import as the main PR with proper
multi-platform validation per the PR body's own "not covered" list.

## What I learned

When a PR's diff size exceeds GitHub's 300-file API limit, that's the
review tool telling you the PR is no longer reviewable — the right
response is to split, not to find a way around the limit. The
workspace-exclusion mechanism (npm ≥ 7 negation pattern) is the only
correct primitive for "two package managers in one tree", but it's
fragile to npm version drift; pinning the minimum npm version in
`engines` becomes load-bearing once you adopt this pattern.
