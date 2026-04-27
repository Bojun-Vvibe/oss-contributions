# sst/opencode#24565 — Refactor npm config handling

- **Head**: `0edbb295d9867ea14258a2c36e56a8f93198b0e4`
- **Size**: +186/-254 across 12 files
- **Verdict**: `merge-after-nits`

## Context

Three independent-but-related cleanups bundled in one PR:

1. Extract `loadOptions(dir)` — the `@npmcli/config` initialization with
   `definitions/flatten/nerfDarts/shorthands` — out of `packages/core/src/npm.ts`
   into a new dedicated `packages/core/src/npm-config.ts` module.
2. Replace the spawn-based `viewLatestVersion` (which shelled out to
   `npm view` → `pnpm view` → `bun pm view`) with a direct HTTP lookup
   against the npm registry, gated on the new `NpmConfig.registry(dir)`
   helper.
3. Drop the now-unused `Npm.outdated` interface entirely from the
   service contract (`Service`, `Interface`, top-level `outdated` export,
   and the `CrossSpawnSpawner` provide chain that only existed to power it).

Plus a focused test file for the new `npm-config` module and a shared
`tmpdir` test fixture.

## Design analysis

### Extraction is clean

`packages/core/src/npm-config.ts:1-40` is a textbook module split: the
`@npmcli/config` boilerplate (the `npmPath`, `argv`, `execPath`, `platform`,
the four `definitions/flatten/nerfDarts/shorthands` imports, `warn: false`)
all moves verbatim into one place, with only the error semantics changing
in a sensible direction:

```ts
catch: (cause) => cause,
}).pipe(Effect.orElseSucceed(() => ({}) as Record<string, unknown>))
```

Old code threw `InstallFailedError` from `loadOptions`; new code swallows
the error and returns an empty config. That's correct for the new caller
in `Npm.reify` (we still want install to proceed with default config if
`.npmrc` parsing barfs) and the new `NpmConfig.registry` caller (which
wants a default registry fallback). Worth a unit test that exercises the
swallow path — currently `npm-config.test.ts` only covers the happy path
(`reads registry from project .npmrc`).

### The `viewLatestVersion` replacement is the right direction

Before, `viewLatestVersion` ran:

```ts
runView(["npm", "view", pkg, "dist-tags.latest", "--json"]).pipe(
  Effect.catch(() =>
    runView(["pnpm", "view", pkg, "dist-tags.latest", "--json"]).pipe(
      Effect.catch(() => runView(["bun", "pm", "view", pkg, "dist-tags.latest", "--json"])),
    ),
  ),
)
```

This had three real problems:

1. **Three separate child-process spawns on every staleness check.** Cold
   spawn of `npm` is ~150–300ms on macOS. Falling through to `pnpm` and
   `bun pm` on a host that has none of them installed (CI minimal images,
   Alpine containers) means three failed exec attempts per check.
2. **Depends on a package manager being on `PATH`.** Bundled binaries
   that ship without a package manager would simply fail this check.
3. **The fall-through chain hid auth failures.** A misconfigured private
   registry would `npm view` fail with auth → `pnpm view` fail with auth
   → `bun pm view` fail with auth — and the whole thing returns a generic
   "Failed to run" string. The user sees "Failed to run" with no signal
   about which manager or why.

Replacing it with a direct HTTPS GET to `${registry}/${name}` parsed for
`dist-tags.latest` removes all three. The registry-resolution path goes
through `NpmConfig.registry(dir)`, which means `.npmrc` overrides
(scoped registries, alternate hosts) are still honored — so this is not
a regression for users on private registries.

The diff doesn't show the new HTTP lookup itself (only the removal in
`npm.ts:97-114`); reviewing the call site to confirm it correctly handles
`401`/`404`/`5xx` distinctly is worth a follow-up read. PR body claims
`bun test test/npm.test.ts` passes, which suggests the existing version
check still works; would be cleaner if a new test pinned the registry-only
path explicitly.

### Removing `outdated` from `Interface` is the right call

```ts
-  readonly outdated: (pkg: string, cachedVersion: string) => Effect.Effect<boolean>
```

If no caller uses it, drop it. Service interfaces should not carry
"reserved for future use" methods. The single in-tree caller has
been migrated (the upgrade flow now uses the registry-direct lookup),
which the diff implies via the removed `Layer.provide(CrossSpawnSpawner.defaultLayer)`
on `defaultLayer` (the spawner was only there to power `outdated`).

### Test fixture refactor

`packages/core/test/fixture/tmpdir.ts:1-13` introduces an
`await using` resource pattern:

```ts
return {
  path: dir,
  async [Symbol.asyncDispose]() {
    await fs.rm(dir, { recursive: true, force: true })
  },
}
```

Modern, correct, requires `using` syntax (TS 5.2+ / Bun-native). This is
a nice ergonomic upgrade over manual `try/finally`+`fs.rm` blocks. Worth
checking that the project's `tsconfig` `target` already supports
`Symbol.asyncDispose` — if `target: ES2022` or below, this needs the
disposable-types polyfill (`tslib` provides it, but it's worth a
`bun typecheck` confirmation, which the PR body claims is passing).

## Concerns / nits

1. **Three semantically distinct changes in one PR.** The module split,
   the registry-direct lookup, and the `outdated` removal are each
   reviewable on their own merits. Bundled together, a future bisect of
   "registry-direct lookup broke something" is harder than it needs to
   be. Not a blocker, but the project's other recent PRs (#24553,
   #24544) have stayed surgical.

2. **No test for the swallow-on-error path** in `NpmConfig.load`. Add a
   test like:

   ```ts
   test("falls back to empty config on malformed .npmrc", async () => {
     await using tmp = await tmpdir()
     await Bun.write(path.join(tmp.path, ".npmrc"), "this is\nnot=valid\nini\x00\x00")
     const config = await Effect.runPromise(NpmConfig.load(tmp.path))
     expect(Object.keys(config)).toEqual([])
   })
   ```

   This pins the new error-swallowing behavior so a future refactor that
   re-introduces `InstallFailedError` here (and breaks `Npm.reify` for
   users with broken `.npmrc`) gets caught.

3. **`registry` helper trims a single trailing slash** (`registry.endsWith("/") ? registry.slice(0, -1) : registry`).
   That's fine for the canonical `https://registry.npmjs.org/` case, but
   doesn't normalize multiple trailing slashes (e.g. `https://reg/foo//`)
   or empty-string. A `URL.canParse` validation step would be more
   defensive; minor.

4. **`test:ci` script added to `packages/core/package.json:9`** —
   `mkdir -p .artifacts/unit && bun test --timeout 30000 --reporter=junit ...`
   This is good but mixes infrastructure with the refactor. If it's tied
   to a CI workflow change in the same PR, it's coherent; otherwise it's
   another orthogonal cleanup.

5. **`@npmcli/config` is loaded fresh on every `NpmConfig.load(dir)` call**
   from `Effect.tryPromise`. Memoizing per-dir would be a follow-up if
   the new `registry()` helper becomes hot-path; for the staleness-check
   call rate this isn't an issue.

## Verdict reasoning

Three sound refactors that each individually move the codebase in the
right direction (less spawning, cleaner module boundaries, smaller
service interface). The combined PR is a bit dense for a single review
and is missing one targeted test (swallow-on-error), but the structural
changes are correct and `bun typecheck` passes per the PR body. The
"merge-after-nits" verdict reflects "land it, but please add the
fallback test as a follow-up commit before the registry-direct lookup
hits a private-registry user with a broken `.npmrc`".

## What I learned

The "shell out to a package manager to get latest version" pattern is
surprisingly common across CLI tools — it appears in everything from
homebrew formulae update checkers to language-server install scripts.
The win from replacing it with a direct registry HTTPS GET is consistent:
faster (one HTTP round-trip vs cold-start of node + the package manager
binary), more portable (works in containers without `npm` on `PATH`),
and gives sharper error messages. The refactor here is a good template
for similar staleness checks elsewhere — including the `bun pm view`
fall-through pattern, which is itself another instance of the same
anti-pattern.
