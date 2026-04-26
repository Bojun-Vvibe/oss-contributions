---
pr: 10367
repo: cline/cline
sha: 6aa63492a47df65a04fb3a6157c54d07632d2dbb
verdict: needs-discussion
date: 2026-04-26
---

# cline/cline #10367 — chore(deps): bump uuid from 11.1.0 to 14.0.0

- **Author**: app/dependabot
- **Head SHA**: 6aa63492a47df65a04fb3a6157c54d07632d2dbb
- **Size**: 47 diff lines (`package.json` + lockfile).

## Scope

Dependabot lockfile bump of `uuid` from `^11.1.0` to `^14.0.0`. Three major boundaries crossed (11→12→13→14). Lockfile-only on the surface, but `uuid` 13.0 was the ESM-only release and 14.0 reorganized the package's `bin` paths.

## Specific findings

- **uuid 13.0 dropped CommonJS.** As of 13.0.0, `uuid` is ESM-only — there is no `dist/cjs` build, only `dist/esm`. If any code in the cline tree (or any *transitive* dep that bundles into cline) does `require("uuid")`, it will throw `ERR_REQUIRE_ESM` at runtime. Cline itself is largely ESM/TypeScript, but VS Code extension hosts have historically been CommonJS bridges. Verify with `node --print 'require("uuid")'` against a built bundle and against any vendored `node_modules` shipped in the VSIX.
- **uuid 14.0 reorganized `bin` paths** — the CLI entry that used to live at `dist/esm/bin/uuid` was moved to `dist-node/bin/uuid` (or similar reshuffle; verify against the 14.0 CHANGELOG). If anything in cline's build/test scripts invokes `npx uuid` or shells out to the CLI, it will break. `grep -r "uuid" package.json scripts/ .github/` to confirm.
- **`crypto.randomUUID()` is the right zero-dep replacement** if cline's Node target is ≥ 14.17 / ≥ 19.0 (where it's available without `crypto.webcrypto`). VS Code's bundled Node is well past that line. Question for the maintainer: is `uuid` even needed anymore? If the only consumers want v4 UUIDs, `crypto.randomUUID()` is faster, dependency-free, and identical output. v1/v3/v5 / NIL UUID consumers still need the package.
- **Bundle size impact** — uuid 14 with ESM tree-shaking is smaller than 11 with CJS, but only if the bundler (esbuild/webpack/rollup as used by cline) actually tree-shakes. Confirm a production VSIX build still loads and the bundle size hasn't regressed (or improved). A `du -sh out/` before/after comparison in the PR description would be ideal.
- **47 diff lines is suspicious.** A clean uuid bump should be ~10 lines (one `package.json` line, a few `package-lock.json` blocks). 47 lines suggests either multiple lockfile entries (transitive consumers also rolling) or a partially regenerated lockfile. Eyeball the lockfile diff to confirm only `uuid` and its dependents (none, since uuid has no deps) changed; nothing else should have moved.
- **No source-side `import` change in the diff** — the package's named exports (`v4`, `v1`, `parse`, `stringify`, `validate`, `version`, `NIL`) are unchanged across 11→14, so existing call sites should keep working as long as the import resolution succeeds. The risk is at the loader, not the API.

## Risk

Medium. The two real failure modes (CJS loader breakage at runtime in extension host context, and the bin-path move) are both *runtime-only* and won't be caught by `tsc`. If cline's CI doesn't actually build the VSIX and run a smoke test against a real VS Code instance, this bump can land green and break on first user install.

## Verdict

**needs-discussion** — three checks before merge: (1) does the production VSIX build load without `ERR_REQUIRE_ESM` (run a packaged install in VS Code, not just `npm test`); (2) does any script invoke the `uuid` CLI binary (`grep` package.json scripts and CI yamls); (3) given Node ≥ 14.17 ubiquity, is the `uuid` dependency still earning its keep, or can it be replaced by `crypto.randomUUID()` for the v4 call sites? The answer to (3) might make this PR moot — close it and remove the dep entirely.
