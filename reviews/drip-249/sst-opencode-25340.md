# sst/opencode #25340 — fix: ensureTitle falls back to main model when getSmallModel fails

- URL: https://github.com/sst/opencode/pull/25340
- Head SHA: `d090342178337d7e4b06553ac0276a109e669a05`
- Files: `packages/opencode/src/session/prompt.ts` (+3/-2)
- Closes #25344

## Context / problem

`ensureTitle` (the auto-titling path for new sessions) wanted "use the configured small model if available, else fall back to the main model." The original line at `prompt.ts:188-190`:

```ts
: ((yield* provider.getSmallModel(input.providerID)) ??
   (yield* provider.getModel(input.providerID, input.modelID)))
```

is broken in two distinct ways given the Effect-TS + `Option`-returning provider API:

1. If `getSmallModel` raises an Effect failure (e.g. provider has no small model configured and the lookup is modeled as a failure), `yield*` propagates the error past `??` entirely, and the surrounding `t.ignore` in the `forkIn` scope silently swallows it. `mdl` is never assigned a usable value — title stream fails — session title stays as the placeholder.
2. If `getSmallModel` returns `Option.None` (an object, not nullish), the `??` fallback simply doesn't fire because `Option.None` is truthy at the JS level.

Either path leaves the session permanently titled `"New session - ..."`, which matches the user-reported symptom in #25344.

## Design analysis

Fix at `prompt.ts:188-190`:
```ts
: (yield* ((yield* provider.getSmallModel(input.providerID)).pipe(
    Effect.orElse(() => provider.getModel(input.providerID, input.modelID))
  )))
```

This is the right Effect-TS shape:
- `provider.getSmallModel(input.providerID)` is yielded first to unwrap its outer Effect (so we have an `Effect<Model>` representing the small-model lookup).
- `.pipe(Effect.orElse(() => provider.getModel(...)))` is the canonical "on failure, run this other Effect" combinator — handles the failure-channel case (1).
- The `Option.None` truthy-trap (case 2) is implicitly addressed if `getSmallModel` models "no small model" as a failure rather than returning `Option.None` of a model. This is consistent with the rest of the file's use of `provider.getModel` returning the model directly, but the PR description doesn't explicitly nail down which channel (failure vs. `Option`) the provider actually uses for "no small model" — a one-line confirmation would help.

## Risks

- The double `yield*` is correct but visually noisy. If `provider.getSmallModel` ever returns `Effect<Option<Model>>` rather than `Effect<Model, _>`, this fix would silently behave wrong (the `Option.None` would be the success value of the outer Effect, `orElse` would never fire). Worth a type assertion `satisfies Effect.Effect<Model, ...>` on the small-model expression to lock the contract.
- No new test. The bug was a silent-degradation user-facing failure — a regression test that mocks `provider.getSmallModel` to fail and asserts `mdl` is the main model would prevent re-introduction.

## Suggestions

- Add a comment explaining the `Effect.orElse` discipline (`// fall back to main model if small model is not configured / lookup fails`) — the next maintainer will not remember why `??` was wrong here.
- Add the regression test described above.

## Verdict

`merge-after-nits` — correct one-line fix for a real silent-degradation bug in the auto-titling path, replaces JS-level `??` (which is wrong against both the failure channel and `Option.None`) with the Effect-TS-native `Effect.orElse` combinator. Wants a comment + regression test + author confirmation of the small-model "no value" channel.
