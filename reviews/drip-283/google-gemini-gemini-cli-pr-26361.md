# google-gemini/gemini-cli PR #26361 — fix(core): externalize https-proxy-agent to fix proxy support

- Head SHA: `6c0d6808edbf3d23d0c3ad39ccedc2768029ae74`
- Size: +14 / -0, 3 files

## Specific refs

- `esbuild.config.js:67` — adds `'https-proxy-agent'` to the `external` array, alongside existing `@google/gemini-cli-devtools` and `@github/keytar`. Tells esbuild to leave the import path untouched so Node resolves it at runtime via native CJS interop.
- `package.json:146` — adds `"https-proxy-agent": "^7.0.6"` as a top-level dependency. Required because once externalized, the package must be present in `node_modules` at install time (not just bundled). Caret range is appropriate — gaxios pins compatible major.
- `scripts/copy_bundle_assets.js:128-138` — copies `https-proxy-agent` and its transitive deps (`agent-base`, `debug`, `ms`) into `bundle/node_modules/` for SEA / single-binary distribution. Uses `cpSync(..., { recursive: true, dereference: true })` — `dereference` is correct for SEA where you can't follow symlinks at runtime.

## Assessment

Correct diagnosis of a real, well-understood esbuild interop issue. `splitting: true` + `format: 'esm'` + dynamic `import()` of a CJS module is a known footgun: esbuild emits an ESM facade chunk that loses named exports, so `(await import('https-proxy-agent')).HttpsProxyAgent` becomes `undefined`. The exact triggering call is cited from gaxios. The "externalize and let Node handle it" pattern is the standard fix and the same approach already used for `keytar` and the devtools package — pattern consistency is a strong signal here.

The transitive-deps list (`agent-base`, `debug`, `ms`) for SEA copy is exactly what `https-proxy-agent@7` needs and matches its declared deps. One concern: there's no test for this. Proxy paths are notoriously hard to unit-test, but a lightweight build-time assertion that `bundle/index.js` does not inline `https-proxy-agent` (e.g. grep for `HttpsProxyAgent` definition outside `bundle/node_modules/`) would catch a regression if `external` is later reverted. Also worth verifying the dep is recorded in `dependencies` rather than `devDependencies` — diff confirms `dependencies`, correct.

Validation in the PR is manual (requires real proxy). Author has only checked the Linux/`npm run` box on the matrix. Maintainers should confirm SEA / Docker variants before tagging.

verdict: merge-after-nits
