# sst/opencode #24819 — fix(npm): resolve cached package name from install dir for non-registry specs

- PR: https://github.com/sst/opencode/pull/24819
- Head SHA: `6555161924299420af4a10e12dc8eb0b305e0f3a`
- Author: sjawhar
- Files: `packages/core/src/npm.ts` (+20/−4), `packages/opencode/test/npm.test.ts` (+114/−0)

## Observations

- Root-cause is correctly localized to the cache fast-path at `packages/core/src/npm.ts:187-199`. The `name` IIFE collapses to the spec string for any non-registry spec because `npa(pkg).name` returns `undefined` for `https://`, `git+https://`, `github:owner/repo`, and `file:` forms — `npa` only knows how to extract `name` from registry-shaped specs. The PR's table enumerating the 4 broken cells vs the 2 working ones is an accurate matrix.
- Fix shape: read the install-root `package.json` (which Arborist authored on the *first* run's fresh-install path) and recover the actual installed package name from the first key of `dependencies`. This is the same source `Npm.install` already consults at `npm.ts:247-275` for dirty-install detection, so the PR is reusing an existing, already-tested invariant — good single-source-of-truth shape.
- Fallback chain `install-root package.json first dep → npa(pkg).name → spec string` is the right precedence: prefer the ground truth Arborist wrote, fall back to `npa` for registry specs (where Arborist's metadata is redundant with `npa`'s parse), and only fall back to the spec string itself if both fail. That last arm preserves backward compat for the unforeseen "`package.json` missing or has no deps" case rather than throwing.
- The fresh-install path is byte-identical to before — only the cache fast-path's `name` resolution moves. This is the right blast radius for a fix to a latent bug: don't perturb the working code path.
- Test surface: 5 new tests covering each non-registry spec form (https tarball, git+https, github shorthand, file path) plus a registry-package regression cell. Each test seeds a synthetic populated cache mimicking what Arborist writes after a successful install, then asserts `Npm.add(spec).directory` resolves to `<cache>/<sanitized>/node_modules/<actualPkgName>/`. Without the fix, 4/5 fail (the registry one passes via the `npa` arm) — this is the right "fix-pins-bug" test discipline. Total file: 11 pass, 0 fail.

## Risks / nits

- `npa` is the package canonically known as `npm-package-arg`; I'd verify the imported binding is still `default` — recent versions changed export shape, and a silent named-import regression would re-introduce the bug.
- The PR description says "first `dependencies` entry" — if Arborist ever writes the root `package.json` with multiple top-level deps (e.g. transitive promotion in a future Arborist version), the "first key" heuristic becomes order-dependent on JSON-object iteration. Worth narrowing to the entry whose `version`/`resolved` field matches the originally requested spec, or asserting `Object.keys(deps).length === 1` and falling through if not.
- No test for the "package.json present but `dependencies` empty/missing" path — this is the second-arm fallback and should have a cell asserting it falls through to `npa(pkg).name` cleanly rather than crashing on `Object.keys(undefined)`.
- The PR title says "for non-registry specs" but the fix arguably applies *more* broadly: any time `npa.name` doesn't agree with what Arborist actually wrote on disk (e.g. registry package with a name redirect) the cache path was previously wrong. Worth widening the description.

## Verdict: `merge-after-nits`

**Rationale:** Right diagnosis, right fix shape, right blast radius, right test discipline. Reuses an existing read from the same source (`Npm.install`'s dirty-install check at `npm.ts:247-275`) so the new path is already de-risked by production traffic on the install side. Small nits worth addressing: tighten "first dep" heuristic against future Arborist behavior, add a cell for empty-`dependencies` fallback, and consider widening the title to describe the general invariant (cache fast-path must read the on-disk truth, not the parsed spec).
