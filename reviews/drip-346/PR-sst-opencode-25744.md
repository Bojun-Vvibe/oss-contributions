# sst/opencode#25744 — fix(npm): resolve cached package name from install dir for non-registry specs

- PR ref: `sst/opencode#25744` (https://github.com/sst/opencode/pull/25744)
- Head SHA: `42bc5869683da5d1a4261e540a81efe62daf077b`
- Title: fix(npm): resolve cached package name from install dir for non-registry specs
- Verdict: **merge-after-nits**

## Review

The diagnosis in `packages/core/src/npm.ts:112-150` is exactly right: `npa(spec).name`
returns `undefined` for tarball URLs, `git+https://`, `github:` shorthand, and `file:`
specs, and the previous code happily passed that `undefined` into
`resolveEntryPoint(name, ...)` on a cache hit. Recovering the real name by reading the
install-root `package.json` and pulling the first `dependencies` key — the same shape
that Arborist writes during the fresh-install path further down — is the cleanest fix
that doesn't require a second source of truth.

I like that the cache-but-no-parseable-`package.json` branch falls through to the
Arborist install path instead of returning a broken entrypoint
(`packages/core/src/npm.ts:130-138`). That preserves recoverability for partially
written caches, which is the scenario most likely to show up in the wild after a
crashed install.

Two nits before merging:

1. `Object.keys(deps)[0]` is non-deterministic if a future change ever writes more
   than one dep into a single install root. Today the install-root only ever holds
   one dep so this is fine, but a one-line comment at
   `packages/core/src/npm.ts:127` to that effect would prevent a future contributor
   from "fixing" the loop into something that picks the wrong package.
2. The new test in `packages/opencode/test/npm.test.ts` covers the cache-hit and
   re-install paths but does not cover the new "cache exists, `package.json`
   present, no `dependencies` key" branch. Worth one assertion that this case
   re-reifies cleanly rather than silently returning a broken entrypoint — that's
   exactly the regression this PR is trying to prevent.
