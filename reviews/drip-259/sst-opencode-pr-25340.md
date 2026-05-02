# sst/opencode PR #25340 — fix: ensureTitle falls back to main model when getSmallModel fails

- PR: https://github.com/sst/opencode/pull/25340
- Head SHA: `d090342178337d7e4b06553ac0276a109e669a05`
- Author: @per-hap-s
- Closes: #25344
- Size: +3 / -2

## Summary

The original line at `packages/opencode/src/session/prompt.ts:188` used `??` to fall back from `getSmallModel` to `getModel`. Two failure modes leaked through:

1. If `provider.getSmallModel(...)` is an Effect that fails, `yield*` propagates the error past `??` entirely, and the failure is later swallowed by the outer `t.ignore` on the forked title-generation scope. Result: `mdl` is unbound, the LLM stream throws, and the session title stays at `"New session - …"` forever.
2. If `getSmallModel` returns `Option.None` (object — truthy), `??` does not engage at all, so the fallback path is dead code.

The fix wraps the small-model effect in `.pipe(Effect.orElse(() => provider.getModel(...)))` and yields the recovered effect. That covers both Effect-failure and Option-None paths.

## Specific references from the diff

- `packages/opencode/src/session/prompt.ts:189-191` — replaces `(yield* getSmallModel(...)) ?? (yield* getModel(...))` with `yield* (yield* getSmallModel(...)).pipe(Effect.orElse(() => getModel(...)))`. Effect-TS-correct.

## Verdict: `merge-after-nits`

Diagnosis is correct, fix is minimal (3-line change), and it's the right primitive to use. Two small things keep this off `merge-as-is`.

## Nits / concerns

1. **Double `yield*` is non-obvious.** `yield* ((yield* getSmallModel(...)).pipe(Effect.orElse(...)))` — the inner `yield*` runs the small-model effect to obtain the value-or-Option, then the result is piped into `orElse` and yielded again. Reviewers will trip on this. Consider:
   ```ts
   const mdl = yield* provider.getSmallModel(input.providerID).pipe(
     Effect.orElse(() => provider.getModel(input.providerID, input.modelID)),
   )
   ```
   which is functionally identical and reads top-to-bottom.
2. **No regression test.** The PR description mentions manual verification but adds no test for either the Effect-failure or `Option.None` path. Given this regressed silently once, a unit test mocking `getSmallModel` to fail (and asserting the title gets generated via `getModel`) would prevent it from breaking again.
3. **`Option.None` claim deserves a code comment.** The reasoning that `Option.None` is object-truthy and thus broke `??` is the most surprising part of this fix. One inline comment at `:189` would help the next reader understand *why* `orElse` is required, not just that it is.
