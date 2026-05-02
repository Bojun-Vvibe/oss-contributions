# google-gemini/gemini-cli PR #26361 — fix(core): externalize `https-proxy-agent` to fix proxy support

- **Head SHA**: `6c0d6808edbf3d23d0c3ad39ccedc2768029ae74`
- **Scope**: `esbuild.config.js`, `package.json`, `scripts/copy_bundle_assets.js`

## Summary

Restores HTTPS proxy support that was broken when the bundler inlined `https-proxy-agent`, which loads its dependency tree via dynamic `import()` and thus must remain external. This PR:

1. Adds `'https-proxy-agent'` to esbuild's `external` array (`esbuild.config.js:67`).
2. Promotes `https-proxy-agent` from devDependency to runtime `dependencies` (`package.json:146`).
3. Copies `https-proxy-agent` plus its transitive deps (`agent-base`, `debug`, `ms`) into `bundle/node_modules/` at build time (`scripts/copy_bundle_assets.js:128-136`).

## Comments

- The dep list `['https-proxy-agent', 'agent-base', 'debug', 'ms']` is hard-coded and will silently rot if `https-proxy-agent` adds a new transitive dep in a future minor. A safer approach: walk `node_modules/https-proxy-agent/package.json`'s `dependencies` recursively, or use `npm ls https-proxy-agent --parseable` at build time. Not a blocker, but worth a TODO comment.
- `dereference: true` on `cpSync` is correct for handling pnpm/yarn symlinked layouts.
- Caret range `^7.0.6` is fine for runtime; matches what was previously a transitive resolution.
- No tests added — proxy resolution is hard to unit-test in CI without a real proxy, but a smoke test that requires the bundled `https-proxy-agent` from `bundle/node_modules` and asserts it loads would catch regressions of the copy step.
- Verify the previous devDependency entry was removed (not visible in this hunk) to avoid duplicate declarations.

## Verdict

`merge-after-nits` — add a TODO/comment about the hard-coded transitive dep list, and confirm the old devDependency entry is gone.
