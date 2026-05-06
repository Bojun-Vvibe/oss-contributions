# google-gemini/gemini-cli#26551 — fix: externalize https-proxy-agent in bundle

- PR: https://github.com/google-gemini/gemini-cli/pull/26551
- Head SHA: `b3acaec3e2b92a3dc0da3235a0bf4732c4d55a2f`
- Size: +3/-0
- Verdict: **merge-after-nits** (man)

## Shape

Three-line change adding `'https-proxy-agent'` to the esbuild `external` array
at `esbuild.config.js:67`, plus a top-level dependency entry in `package.json`
at `:146` (`"https-proxy-agent": "^7.0.6"`) and the corresponding
`package-lock.json` registration at `:14`. The intent is to keep
`https-proxy-agent` out of the bundled CLI artifact and load it from
`node_modules` at runtime instead — necessary when downstream code does
conditional `require('https-proxy-agent')` based on env, since esbuild's
default behavior would either tree-shake it or attempt to inline transitive
deps that aren't safe to bundle (e.g., the `agent-base` chain).

## Notable observations

- Pinning to `^7.0.6` (caret semver) is reasonable since the v7 line is the
  current major and the package is mature; verify there's no `peer` dep
  conflict with whichever transitive consumer (likely `proxy-agent` or
  `node-fetch`) was previously providing it implicitly.
- Adding to `dependencies` (not `devDependencies` or `optionalDependencies`)
  is correct — when you mark a module `external` in esbuild, it MUST be
  resolvable in the published `node_modules` tree at runtime, otherwise
  every user hits `Cannot find module 'https-proxy-agent'` on first proxy
  use.
- The neighboring `external` entries (`@lydell/node-pty-win32-x64`,
  `@github/keytar`, `@google/gemini-cli-devtools`) are all
  native/platform-bound packages — `https-proxy-agent` is pure JS, so the
  motivation is different (likely either bundle-size regression or a
  conditional-require pattern in a transitive consumer). The PR body should
  explicitly call out *which* consumer is doing the conditional require so
  future maintainers don't tree-shake the entry away again on cleanup.

## Concerns / nits

- No test added. A single integration test that sets `HTTPS_PROXY` and asserts
  the CLI process can `require('https-proxy-agent')` from the published bundle
  layout would lock the contract. Without it, the next bundler refactor will
  silently break proxy support.
- Worth adding a `// keep external — required by <consumer> conditionally` 
  comment next to the new entry in `esbuild.config.js:67`. The other entries
  are self-evidently external (native bindings); this one isn't.
- The lockfile entry adds the package at the workspace root — confirm there's
  no version conflict with a nested workspace (`packages/*`) that pulls a
  different range transitively.
