# sst/opencode #24374 — fix(sdk): load cross-spawn through require

- **Repo**: sst/opencode
- **PR**: #24374
- **Author**: pascalandr (Pascal André)
- **Head SHA**: b6286471df4ff18257df387896873facd4739bec
- **Link**: https://github.com/sst/opencode/pull/24374
- **Closes**: #22281
- **Size**: 6 lines changed, two near-identical edits to
  `packages/sdk/js/src/server.ts` and `packages/sdk/js/src/v2/server.ts`.

## What it changes

Replaces the ESM default-import of `cross-spawn` with a
`createRequire(import.meta.url)("cross-spawn")` call, casting the
result to the imported type via `import type launchType from
"cross-spawn"`.

Before (`server.ts:1`, `v2/server.ts:1`):
```ts
import launch from "cross-spawn"
```

After (`server.ts:1-6`, `v2/server.ts:1-6`):
```ts
import { createRequire } from "node:module"
import type launchType from "cross-spawn"
import { type Config } from "./gen/types.gen.js"
import { stop, bindAbort } from "./process.js"

const launch = createRequire(import.meta.url)("cross-spawn") as typeof launchType
```

## What was actually broken

`cross-spawn` is a CommonJS package (`module.exports = function
spawn(...)`). When loaded via Node's ESM loader, the TS-emitted
`import launch from "cross-spawn"` works because tsc/esbuild
synthesizes a default export from `module.exports`. Bun's raw ESM
loader (in some configurations — typically when the SDK is
installed as a *transitive* dependency by a plugin host that
re-resolves the module) doesn't synthesize that default. Result:
`launch` is `undefined`, and `launch(...)` throws
`TypeError: launch is not a function` at process spawn time.

`createRequire` sidesteps the synthesis entirely — it returns
`module.exports` directly, regardless of the loader.

## Strengths

- **Surgical and correct.** This is exactly the canonical fix for
  the CJS-default-export-under-ESM problem. The pattern is
  documented in Node's own `node:module` docs and is what major
  libraries (e.g. `chokidar`, `yargs`) use in their hybrid
  ESM/CJS builds.
- **Type-only import preserved.** `import type launchType` keeps
  the `typeof launchType` cast precise — runtime still loads via
  `require`, but TypeScript still sees the proper signature.
  Without that, the `as typeof launchType` cast would have to be
  `as (...args: any[]) => ChildProcess` or similar.
- **Both `server.ts` and `v2/server.ts` updated symmetrically.**
  Easy to forget the v2 copy; the diff catches both.
- **Verification was actual** — PR body cites `bun typecheck`
  from `packages/sdk/js` plus `bun ./script/build` (truncated in
  the view but referenced).

## Concerns / asks

- **No regression test.** The bug was about a *runtime loader
  shape*, not a type, so `bun typecheck` wouldn't have caught
  it. A test that spawns the server via the SDK from a Bun
  runtime and asserts the process actually starts would close
  the regression for good. That said, this is hard to set up in
  a unit-test environment and was likely tested manually.
- **No comment explaining *why*.** Six months from now, someone
  refactoring imports will see two `createRequire` calls for one
  package and "tidy up" by switching them back to ESM imports,
  reintroducing the bug. A one-line comment like `// cross-spawn
  is CJS-only; Bun's ESM loader doesn't synthesize default
  exports — see #22281` next to the require call would prevent
  that.
- **The two files diverge only in the `import { stop, bindAbort
  } from "./process.js"` vs `from "../process.js"` path.** If
  there are more such `cross-spawn` consumers in the SDK
  (`packages/sdk/js/src/**/*.ts`), they'd hit the same bug.
  Worth a `grep -r "from \"cross-spawn\"" packages/sdk/js/src`
  to make sure these are the only two.
- **`createRequire(import.meta.url)` is called *every time the
  module is imported*, but the result is stored in a
  module-level `const`** so it only runs once. Fine, but worth
  noting that if either file is dynamically imported in a tight
  loop (which the SDK shouldn't do), this becomes a real cost.
  Probably a non-issue.

## Verdict

**merge-as-is** — minimal, correct, well-targeted fix for a
real Bun + ESM + CJS shape mismatch. The asks (comment, grep
for other consumers) are nice-to-haves and shouldn't gate the
merge. The fix has been used in production by many other
TypeScript libraries with the same Bun + CJS issue.

## What I learned

The `import default from "cjs-pkg"` synthesis is *not*
guaranteed by the ESM spec — it's a Node-loader convenience.
Bun, Deno, and the browser-side bundlers each handle it
differently. Whenever a package ships only `module.exports = ...`
(no named ESM exports), the safe pattern under ESM is
`createRequire(import.meta.url)(name)`, with a `import type`
sidecar for typing. The TS-side `esModuleInterop` flag papers
over the difference at the type level, but doesn't help at
runtime under non-Node loaders. Worth keeping in mind for any
SDK that's expected to run in mixed runtimes.
