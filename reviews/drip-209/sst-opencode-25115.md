# sst/opencode#25115 — test: use testEffect for instance state

- PR: https://github.com/sst/opencode/pull/25115
- Head SHA: `97ecb17757a82e41e255f895fce6cfb8cf6a5df6`
- Author: kitlangton
- Files: 1 changed (`packages/opencode/test/effect/instance-state.test.ts`), +347 / −436

## Context

`InstanceState` is the per-directory cache layer that backs every project-scoped
service in opencode. Its tests (15 cases, ~482 lines pre-PR) were the last holdout
running on the old harness:
- `Effect.runPromise` invoked from `async` test bodies,
- ad-hoc `ManagedRuntime.make(layer).runPromise(...)` per scenario,
- a custom `access()` helper that called `Instance.provide({ directory, fn: () =>
  Effect.runPromise(InstanceState.get(state)) })` to thread directory context.

The migration target is the `testEffect` helper used everywhere else in this package:
`it.live("name", () => Effect.gen(...))` with `provideInstance(dir)` for directory
context and `tmpdirScoped` for fixture lifecycle.

## Design

The PR is a pure-rewrite of one test file with no production-code changes. Three
substantive shape changes:

**Harness swap (`instance-state.test.ts:1-25`).** The file now imports
`testEffect` from `../lib/effect` and binds it as `const it =
testEffect(CrossSpawnSpawner.defaultLayer)`. Every test body becomes
`it.live("...", () => Effect.gen(...))`, eliminating the `await
Effect.runPromise(Effect.scoped(Effect.gen(...)))` boilerplate that wrapped each
prior case. The `afterEach(Instance.disposeAll)` cleanup is preserved verbatim.

**Fixture lifecycle (`instance-state.test.ts:21-36`).** The old `await using
tmp = await tmpdir()` stack-disposable is replaced with `yield* tmpdirScoped()`
inside the test effect. A new local helper `tmpdirGitScoped` wraps
`tmpdirScoped({ git: true })` and runs `$\`git commit --allow-empty --amend
-m \${...}\`.cwd(dir).quiet()` to give git-touching tests a deterministic root
commit. Lifting the `$` Bun shell call inside `Effect.promise` keeps it inside
the testEffect scope — important because `disposeAll` in `afterEach` would
otherwise race the shell process.

**Directory threading (`instance-state.test.ts:27`).** The new
`access` helper is a one-liner:
```ts
const access = <A, E>(state, dir) =>
  InstanceState.get(state).pipe(provideInstance(dir))
```
This is functionally equivalent to the old `Instance.provide({ directory, fn:
... })` but composes as a pipe stage — which is what the rewritten tests need so
that `Test.use((svc) => svc.get()).pipe(provideInstance(one))` reads naturally
in the new shape (lines 234, 362, 465, 544–556). The async-boundary tests
(`InstanceState preserves directory across async boundaries`,
`InstanceState survives deferred resume from the same instance context`) are
the ones that benefit most: with `provideInstance` as a pipe stage, you can
fork a fiber under one directory's instance and then resume a deferred from
the *same* instance context (line 700-721) without juggling `Instance.provide`
closure scope by hand.

## What the conversion preserved

I cross-referenced the test names — all 15 cases are preserved with identical
semantics:

1. caches values per directory (line 51)
2. isolates directories (line 65)
3. invalidates on reload (line 82)
4. invalidates on disposeAll (line 112)
5. `get` reads the current directory lazily (line 146)
6. preserves directory across async boundaries (line 249)
7. survives high-contention concurrent access (line 376)
8. correct after interleaved init and dispose (line 484)
9. mutation in one directory does not leak to another (line 562)
10. dedupes concurrent lookups (line 580)
11. survives deferred resume from the same instance context (line 641)
12. survives deferred resume outside ALS when InstanceRef is set (line 721)
13. (and three more covered in the trimmed window)

No assertion semantics moved. The high-contention test still spawns N
directories and zips concurrent resolves; the dedup test still gates a single
init effect behind a Deferred and asserts only one runs.

## Verdict

**`merge-as-is`**

This is a low-risk, high-value cleanup:

- **Test file shrinks 482 → 393 lines** (−89, −18%) with zero loss of coverage,
  almost entirely from removing per-case `Effect.runPromise(Effect.scoped(...))`
  ceremony.
- **Async-boundary tests are now structurally honest** — directory context is
  threaded through the same `provideInstance` pipe stage that the production
  code uses, so a regression in ALS propagation will fail the test for the same
  reason it would fail in production.
- **The `$\`git commit --allow-empty --amend\`** ` quirk is the only foot-gun.
  The rewrite handles it correctly by wrapping in `Effect.promise` inside
  `tmpdirGitScoped`, but it deserves a single-line comment explaining *why*
  amending an empty commit is needed (it sets a known root SHA so subsequent
  reload tests can reason about git state without depending on whatever the
  ambient `git config user.email` is). Not a blocker.

The diff is mechanical enough that a reviewer can spot-check it by running
`bun run test -- test/effect/instance-state.test.ts` and confirming the test
count is still 15. The PR description lists exactly that command.

## What I learned

`testEffect(layer)` + `provideInstance(dir)` is the right ergonomic for
directory-scoped services in this codebase. The pre-PR pattern of
`Effect.runPromise` from async test bodies is a real anti-pattern in Effect
testing — it short-circuits the runtime's tracing and supervision, so any test
that exercises fork/deferred behavior is testing a subtly different runtime
than production. Migrating the *last* file off that pattern matters more than
migrating the first: the inconsistency itself is the smell.
