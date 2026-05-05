# google-gemini/gemini-cli PR #26551 — fix: externalize https-proxy-agent in bundle

- URL: https://github.com/google-gemini/gemini-cli/pull/26551
- Head SHA: `b3acaec3e2b92a3dc0da3235a0bf4732c4d55a2f`
- Size: +3 / -0 (esbuild config + package.json + lockfile)

## Summary

Three-line packaging fix: marks `https-proxy-agent` as `external` in the esbuild config so it is not bundled into the published artifact, and adds it as a top-level runtime `dependency` in `package.json` (and the corresponding `package-lock.json` entry) so npm/pnpm/yarn install it at install time. This is the standard "bundle excludes native or quirky modules; package.json declares them so the resolver finds them at runtime" pattern.

## Specific findings

- `esbuild.config.js:67` — adds `'https-proxy-agent'` to the `external` array, alongside neighbors `@lydell/node-pty-win32-x64`, `@github/keytar`, `@google/gemini-cli-devtools`. Position in the array is alphabetically reasonable. Effect: esbuild will leave `require('https-proxy-agent')` unrewritten in the output bundle and node will resolve it via `node_modules` at runtime.
- `package.json:146` — adds `"https-proxy-agent": "^7.0.6"` to `dependencies`. Caret-major-7 range is appropriate (7.x is the current stable line; 8.x would be a breaking-change opt-in).
- `package-lock.json:14` — corresponding lockfile entry added. (Lockfile diff is one line in the PR view; full transitive resolution presumably included but truncated from the displayed diff.)

## Notes

- The PR title says "externalize" but doesn't explain *why* externalization was needed. Common reasons: (a) `https-proxy-agent` does dynamic `require()` of `agent-base` that esbuild's tree-shaker mishandles, (b) the bundled copy was being preferred over a user-installed copy, breaking proxy-config detection, (c) the bundle was too large. Worth a one-line PR description / commit message naming the actual symptom (probably "proxy connections fail when bundled" or "duplicate copies of agent-base cause `instanceof` checks to fail"). Not a merge blocker but valuable for future archaeologists.
- No test. This is a build-output behavior change that's hard to unit-test; the right validation is "run a bundled `gemini` against a known HTTPS_PROXY and confirm the request egresses through it." Maintainer should manual-verify before merge.
- The choice of `^7.0.6` as the floor implies `https-proxy-agent` is already being pulled in transitively at >= 7.0.6 elsewhere in the dep tree. Worth checking the lockfile to confirm there isn't now a *second* copy being installed at a higher version, which would defeat the point.

## Verdict

`merge-after-nits`
