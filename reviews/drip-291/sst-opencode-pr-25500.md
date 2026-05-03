# sst/opencode PR #25500 — fix(opencode): exclude .map files from CLI binary

- URL: https://github.com/anomalyco/opencode/pull/25500
- Head SHA: `f21543678c54adf12f7cdac460b5bd00d45adad9`
- Author: PanAchy
- Repo: sst/opencode

## Summary

One-line filter added to `createEmbeddedWebUIBundle` in
`packages/opencode/script/build.ts` so that `.map` (source map) files emitted
by the embedded Web UI build are dropped before they get baked into the CLI
binary as embedded imports. Net diff: +1 / -0.

## Specific notes

- `packages/opencode/script/build.ts:64` — the new
  `.filter((file) => !file.endsWith(".map"))` sits between the path
  normalization `.map(file => file.replaceAll("\\", "/"))` (line 63) and the
  `.sort()` (line 65). Order is correct: paths are already normalized and
  there is nothing downstream that depends on `.map` files appearing in
  `imports` (line 66).
- The change does not touch the build configuration of the Web UI itself, so
  source maps are still generated on disk; they're only excluded from the
  embedded blob. That keeps debugging the standalone Web UI build unaffected
  and only shrinks the CLI binary, which is the stated goal.
- No tests exist for this script; given the change is a one-token
  endsWith filter, manual verification (smaller binary, missing `.map`
  entries in the generated import manifest) is sufficient.
- Minor nit: would be slightly more robust to also filter `.LICENSE.txt`
  or other build-side metadata files in the same place if the goal is "ship
  only runtime assets", but that is scope creep for this PR.

verdict: merge-as-is
