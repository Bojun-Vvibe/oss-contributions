# google-gemini/gemini-cli #26361 — fix(core): externalize https-proxy-agent to fix proxy support

- PR: https://github.com/google-gemini/gemini-cli/pull/26361
- Author: sotokisehiro
- Head SHA: `a0e0ff253153c0e098d34bfc36c92278a1daa2c8`
- Updated: 2026-05-03T04:46:52Z

## Summary
Fixes broken HTTPS proxy support that ships in the bundled CLI. `https-proxy-agent` is dynamically `import()`-ed at runtime, but esbuild was bundling it (or its transitive deps), leaving the runtime resolution path broken in a packaged install. Three coordinated changes: (1) add `'https-proxy-agent'` to the `external` array in `esbuild.config.js:67`, (2) move `https-proxy-agent: ^7.0.6` from devDependencies (presumably) to `dependencies` in `package.json:146`, (3) add a `copy_bundle_assets.js:128-145` block that copies `https-proxy-agent`, `agent-base`, `debug`, and `ms` (the full transitive tree) into `bundle/node_modules/`.

## Observations
- `esbuild.config.js:67`: external addition is alphabetically out of order (comes after `@google/gemini-cli-devtools`). The list looks loosely-grouped but adding to the bottom is consistent with the trailing entries. Minor nit.
- `package.json:146`: moving to `dependencies` is required so consumers who `npm install` the package (not just the bundled binary) also get the proxy agent. Correct.
- `scripts/copy_bundle_assets.js:130-132`: the comment "Keep this list in sync with https-proxy-agent's transitive dependencies. Current tree: https-proxy-agent -> agent-base, debug -> ms" is critical — this list is a hand-maintained mirror of the npm dep graph. **Risk:** if `https-proxy-agent@7.x` adds a new transitive dep (or `debug` does), the bundle silently breaks at runtime when the dynamic import tries to resolve the missing module. Long-term fix would be programmatic resolution (walk `npm ls https-proxy-agent --json`), but the comment + the hard-fail (`process.exit(1)` on missing pkg, line 137-141) is acceptable mitigation.
- `scripts/copy_bundle_assets.js:140`: `cpSync(pkgSrc, pkgDest, { recursive: true, dereference: true })` — `dereference: true` is the right choice for symlinks (turns them into real files in the bundle). Important for npm 10's symlinked node_modules layouts.
- `scripts/copy_bundle_assets.js:137-141`: the `if (!existsSync(pkgSrc))` hard-fail is good. Catches the case where `npm install` was skipped or the package was pruned — better to fail the build than ship a broken bundle.
- **Missing:** a test or a smoke-script that verifies the bundled binary can actually load `https-proxy-agent` at runtime. Without it, the next time someone tweaks `esbuild.config.js` or the bundle script, this fix can silently regress. A trivial `node bundle/cli.js --check-proxy-support` or even a CI step that sets `HTTPS_PROXY=http://localhost:1` and asserts a connection error (proving the agent loaded) would lock in the fix.
- **Missing:** documentation update. If the CLI has a "proxy support" doc page, it should mention this works in bundled installs as of this PR — there are likely existing GitHub issues asking why proxy doesn't work that should be linked in the PR description.
- No security risk in the diff itself. `debug` and `ms` are well-known low-risk packages.

## Verdict
`merge-after-nits`
