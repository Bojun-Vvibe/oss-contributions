# sst/opencode#25354 — core: fix npm package detection for cached dirs without installed packages

- **PR:** https://github.com/sst/opencode/pull/25354
- **Head SHA:** `39204c92dfde884cb308e47381a0033dd31c1dc4`
- **Stats:** +41 / -2 across 2 files (`packages/core/src/npm.ts`, `packages/core/test/npm.test.ts`)
- **Closes:** #24431

## Context
`Npm.add` in `packages/core/src/npm.ts:120-130` uses an existence-check on the cache dir to decide whether the package is already installed. The shape was:

```ts
if (yield* afs.existsSafe(dir)) {
  return resolveEntryPoint(name, path.join(dir, "node_modules", name))
}
const tree = yield* reify({ dir, add: [pkg] })
const first = tree.edgesOut.values().next().value?.to
if (!first) return yield* new InstallFailedError({ add: [pkg], dir })
```

The bug: `dir` exists once the cache directory itself is created (any prior partial install, mkdir-only, or aborted reify creates it). The "is installed" check therefore short-circuits to `resolveEntryPoint` against a `node_modules/<name>` path that doesn't exist, returning `Option.None` for the entrypoint instead of triggering install.

## Design
Two edits at `npm.ts:123-133`:
1. Existence-check is narrowed to `path.join(dir, "node_modules", name)` (the package's own dir under `node_modules`), not the parent cache dir. So "cache dir exists but package wasn't installed" no longer short-circuits.
2. After `reify` returns with no edges (`!first`), instead of going straight to `InstallFailedError`, the code falls through to one more `resolveEntryPoint` attempt against the post-reify tree:
   ```ts
   if (!first) {
     const result = resolveEntryPoint(name, path.join(dir, "node_modules", name))
     if (Option.isSome(result.entrypoint)) return result
     return yield* new InstallFailedError({ add: [pkg], dir })
   }
   ```
   This handles the file-spec case where reify installs the package into `node_modules/<name>` but doesn't surface it as an `edgesOut` entry (the `arborist` tree shape for `file:` specs doesn't always populate edges the same way as registry installs).

## Tests
New `npm.test.ts:46-63` test (`reifies when package cache directory exists without the package installed`) builds the exact dispositive fixture: `mkdir cache/packages/<sanitized-spec>` (creates the directory but installs nothing), then asserts `npm.add(file:...)` returns `Option.isSome(entry.entrypoint)`. The full layered `npmLayer` builder at `:23-29` wires `EffectFlock`, `AppFileSystem`, `Global` (with the test cache dir), and `NodeFileSystem` — heavier than ideal for a unit test but necessary because `Npm.add` depends on all four.

## Risks
- The post-reify second-attempt at `resolveEntryPoint` quietly papers over `arborist`'s `edgesOut`-empty-but-install-succeeded contract; if the underlying library tightens that contract later this fallback becomes a no-op and the original `InstallFailedError` path returns. Add a one-line comment explaining *why* the second attempt exists (the `file:`-spec edge-shape gap) so it doesn't get stripped as "dead code" in a future refactor.
- The narrowed existence check now does an extra `existsSafe` syscall per add (cache dir → `node_modules/<name>`); negligible but worth noting if `npm.add` is hot.
- No negative test for the "cache dir exists *and* package install fails legitimately" path — would prove `InstallFailedError` still fires when both attempts return `None`.

## Suggestions
1. Inline comment at the new fallback explaining the `file:`-spec `edgesOut`-empty case so future contributors understand why the duplicate `resolveEntryPoint` isn't redundant.
2. One negative test asserting `InstallFailedError` is still raised when `reify` actually fails (vs. just empty-edges).
3. Consider extracting the two `path.join(dir, "node_modules", name)` instances into a `const installedPath = path.join(dir, "node_modules", name)` to avoid drift between check and resolve sites.

## Verdict
**merge-after-nits** — correct narrow fix at the right boundary, dispositive test, but a comment on the `arborist`-shape rationale and one negative test would lock the contract.

## What I learned
Existence checks on aggregate paths (cache dirs, working directories) are one of the classic "silent short-circuit" bugs. The directory existing means *something* happened there, not that *the thing you wanted* happened. The right shape is always to check the leaf artifact (`node_modules/<name>`), not the parent. And when the install layer (`arborist` here) returns an empty `edgesOut` set despite a successful install — a known shape for `file:` specs — the right fix is a fallback `resolveEntryPoint` attempt, not a contract change in the upstream library.
