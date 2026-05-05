# QwenLM/qwen-code #3849 — feat(models): add cross-authType model resolution to ModelRegistry and ModelsConfig

- **Head SHA:** `b2719e6a2a33c8ba616604cc8fd863f2c6cfc696`
- **Base:** `main`
- **Author:** B-A-M-N
- **Size:** +517 / −275 across `packages/core/src/{core/client.ts,core/client.test.ts,models/modelRegistry.ts,models/modelRegistry.test.ts,models/modelsConfig.ts,models/modelsConfig.test.ts}`
- **Verdict:** `merge-after-nits`

## Summary

Refactor follow-up to PR #3815. PR #3815 added
`resolveModelAcrossAuthTypes()` as a private method inside
`GeminiClient`. This PR extracts the lookup into the data layer where
it structurally belongs: `ModelRegistry.getModelAcrossAuthTypes(modelId,
preferredAuthType?)` and `ModelsConfig.getResolvedModelAcrossAuthTypes(...)`.
Net result: `client.ts` shrinks (the −275 side of the diff is
overwhelmingly its private helper + tests), and the cross-authType
lookup is now reusable by anything that holds a `ModelRegistry`
reference.

## What's right

- **The extraction is actually motivated.** Pre-PR, the cross-authType
  search lived as a private method in `GeminiClient`. That's wrong on
  two counts: (1) it's data-layer logic — "given a model id, find any
  authType that has it registered" is purely a function of the
  registry's contents, with no client-state dependency; (2) anything
  else needing the same lookup (e.g. config validation, model selector
  UI, settings dump) had to instantiate `GeminiClient` or duplicate the
  iteration. This PR puts it where it belongs.

- **Implementation at `modelRegistry.ts:154-176` is correct and
  idiomatic.** The preferred-authType early-exit (`if (preferredAuthType)
  { const resolved = this.getModel(preferredAuthType, modelId); if
  (resolved) return resolved; }`) at lines 165-168 means the common
  case (caller knows the right authType) doesn't pay for full iteration.
  The `if (authType === preferredAuthType) continue;` at line 173
  prevents re-checking the preferred one — a correctness detail, not
  just a perf thing, because if `getModel` had side effects the double
  call would matter.

- **`Map.keys()` iteration is order-stable in JavaScript.**
  `this.modelsByAuthType.keys()` at `modelRegistry.ts:172` iterates in
  insertion order, which means the cross-authType resolution is
  deterministic given the same registration order. Downstream callers
  that depend on "first match wins" semantics get reproducible
  behavior. Worth a one-line JSDoc note that the iteration order
  matters for consumers.

- **`ModelsConfig.getResolvedModelAcrossAuthTypes` is a thin delegating
  wrapper** to `ModelRegistry.getModelAcrossAuthTypes`. Right shape —
  `ModelsConfig` is the public-facing config façade and shouldn't
  duplicate the registry's iteration logic. The added test in
  `modelsConfig.test.ts` verifies the delegation works through the
  public API surface.

- **`client.ts` diff is largely deletion** — the `−275` lines on
  `client.test.ts` (lines 457-595 and 636-735 in the pre-PR version,
  collapsed in the new version) are the now-redundant tests of the
  private helper. Deleting tests for deleted code is correct; the
  equivalent coverage moves to the new test in
  `modelRegistry.test.ts:530-602` which exercises
  `getModelAcrossAuthTypes` directly.

- **`getResolvedModel` was added at `client.ts:1100-1120` (the +20-line
  hunk)** to give `client.ts` its own thin wrapper that delegates to
  `modelsConfig.getResolvedModelAcrossAuthTypes` while preserving the
  client-level conventions (auth-token resolution, etc.). Composition
  over duplication.

## Nits (worth fixing before merge)

- **Public-API documentation on `getModelAcrossAuthTypes`** at
  `modelRegistry.ts:154-163` is a 4-line JSDoc that says what the
  function does but not the iteration-order guarantee. Since callers
  will now depend on "first registration wins" implicitly, this
  guarantee should be in the contract: `// Iteration follows registration
  order (Map insertion order). When multiple authTypes register the
  same modelId, the first-registered match is returned.`

- **`AuthType` import at `modelRegistry.ts`** — verify the import is
  for the type only (`import type { AuthType } from ...`) rather than a
  value import, since `AuthType` is used purely as a type parameter in
  the new method signature. Tree-shaking + no-cycle concerns.

- **Test for the "preferred authType doesn't have it, but another one
  does" path.** The new tests at `modelRegistry.test.ts:530-602` cover
  the basic "find a model across authTypes" but it's worth pinning
  the tricky case explicitly: caller passes `preferredAuthType=A`,
  modelId is registered only under `B`, expect resolved-from-B. This
  is the path that the early-exit + iteration-skip-A logic at lines
  165-174 specifically handles, and a regression here would silently
  return the wrong model rather than throwing.

- **`client.ts` `resolveModelAcrossAuthTypes` private removal** —
  confirm there are no other callers in the codebase that imported or
  re-exported the old private symbol via type-narrowing tricks. A
  quick grep for `resolveModelAcrossAuthTypes` on the post-PR tree
  should yield zero hits in `packages/core/src/` (test references
  fine).

- **The PR description mentions follow-up to #3815** — worth linking
  in the commit message body (not just the PR description) so
  `git log --grep` from someone investigating cross-authType resolution
  finds both PRs together.

## Risks

- **Behavior should be identical** to the pre-PR private helper —
  same iteration order, same first-match semantics, same return type.
  The risk is purely "did the extraction preserve every call site?"
  which is verifiable from the test diff: tests that previously
  exercised `client.resolveModelAcrossAuthTypes` are replaced by tests
  exercising the new public API, with the same input/output
  expectations.
- **No new public surface beyond the two new methods** — the
  `getModel`/`getModelAcrossAuthTypes` distinction is clean and
  doesn't muddy existing call sites.

## Verdict reasoning

`merge-after-nits`: well-motivated extraction into the right layer,
correct early-exit + skip-preferred logic, test coverage moved with
the code. The nits are documentation polish (iteration-order
contract) and one missing test path (preferred-misses, fallback-hits)
that would harden against future refactors.
