# sst/opencode #25340 — fix: ensureTitle falls back to main model when getSmallModel fails

- **Repo:** sst/opencode
- **PR:** #25340
- **URL:** https://github.com/sst/opencode/pull/25340
- **Head SHA:** `d090342178337d7e4b06553ac0276a109e669a05`
- **Files touched:** `packages/opencode/src/session/prompt.ts` (+3 -2)
- **Verdict:** merge-after-nits

## Summary

`ensureTitle` (the auto-title side-call) previously did
`(yield* getSmallModel(input.providerID)) ?? (yield* getModel(...))`. The
problem: `getSmallModel` returns an Effect, and if that Effect *fails*
(not just returns nullish), the `??` never gets a chance to evaluate the
fallback because the failure short-circuits the surrounding generator.
Providers that don't expose a small/cheap model variant (or that error
on the lookup, e.g. transient registry failure) cause the whole title
generation to crash and bubble up as a session-loop error.

## Specific notes

- **`packages/opencode/src/session/prompt.ts:188-191`** — the new form
  uses `Effect.orElse(() => provider.getModel(input.providerID, input.modelID))`
  so the fallback runs on **failure**, not just on null. This is the
  right semantic: a missing small-model is a real condition (Anthropic
  doesn't ship a "small" variant for every line, etc.).
- The double `yield*` (`yield* ((yield* getSmallModel(...)).pipe(...))`)
  reads awkwardly. Functionally correct, but consider extracting:
  ```ts
  const mdl = ag.model
    ? yield* provider.getModel(...)
    : yield* provider.getSmallModel(input.providerID).pipe(
        Effect.orElse(() => provider.getModel(input.providerID, input.modelID))
      )
  ```
  i.e. drop the inner `yield*` — `Effect.orElse` takes care of the
  composition without an inner generator unwrap.

## Nits

- No test added. The original was untested too, but a 4-line fixture
  with a mock provider whose `getSmallModel` returns
  `Effect.fail(new ProviderError())` would lock this in.
- `Effect.catchAll(() => ...)` may be slightly more idiomatic than
  `Effect.orElse(() => ...)` here since you're explicitly replacing on
  any failure; `orElse` carries an "alternative success" connotation
  that some Effect users find subtly different.
