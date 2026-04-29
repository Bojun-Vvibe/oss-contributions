# sst/opencode#24962 — fix(opencode): apply agent variant when no explicit model is configured

- **Repo:** sst/opencode
- **PR:** [#24962](https://github.com/sst/opencode/pull/24962)
- **Head SHA:** `4c59e2c352d25fe0b3e342669d8cece71f81c70f`
- **Author:** 21pounder
- **Size:** +1 / -1 across 1 file (`packages/opencode/src/session/prompt.ts`)

## Summary

One-line correctness fix in the variant-application gate at
`packages/opencode/src/session/prompt.ts:902`. Closes #21632.

## What's actually going on

The `same` boolean controls whether the agent's configured `variant` (a model
override that swaps in for a specific provider/model pair) gets applied when the
user invokes that agent. Original code:

```ts
const same = ag.model && model.providerID === ag.model.providerID && model.modelID === ag.model.modelID
```

`ag.model` is the agent's *explicit* model declaration in its config — it's
optional. When an agent doesn't declare a model (typical for agents that should
inherit the parent session's or default model), `ag.model` is undefined, so `same`
short-circuits to a falsy value, and the next condition

```ts
const full =
  !input.variant && ag.variant && same
    ? yield* provider.getModel(...)
    : ...
```

never enters the variant branch. Result: an agent with `variant` set but no `model`
set gets its variant *silently ignored* — the bug in #21632.

The fix:

```ts
const same = !ag.model || (model.providerID === ag.model.providerID && model.modelID === ag.model.modelID)
```

Now `same` reads as "the resolved model is consistent with the agent's declared
model, OR the agent has no opinion on the model." Both semantics match the intent
of "apply variant when nothing contradicts it."

## Specific line refs

- `packages/opencode/src/session/prompt.ts:902` — the changed conjunction.
- `packages/opencode/src/session/prompt.ts:903-906` — the `full` block whose
  `ag.variant && same` guard now actually fires for variant-only agents.
- `packages/opencode/src/session/prompt.ts:898` — the `model` resolution chain
  `input.model ?? ag.model ?? lastModel(input.sessionID)` that produces the
  `model` whose providerID/modelID is being compared.

## Reasoning

The diff is correct and minimal. The trinary precedence in the new expression is
unambiguous because of the explicit parens around the AND clause, so there's no
risk of the reader misreading `!a || b && c`. The boolean "no opinion = treat as
match" pattern is exactly the right shape for this kind of optional-config gate.

Two adjacent risks worth eyeballing before merge:

1. **`input.model` interaction.** When `input.model` is set *and* `ag.model` is
   unset, the new logic treats them as `same`, so an explicit per-invocation
   `input.model` will still get the agent's variant applied. That's almost
   certainly correct (variant should apply to the resolved model regardless of
   *how* it got resolved), but worth confirming against the original #21632 repro
   — if the user filed the bug expecting "explicit `--model` should suppress
   variant", this fix doesn't satisfy that.
2. **`lastModel(sessionID)` interaction.** Same situation but for session-scoped
   resume: if `ag.model` is unset and the session resumes a model that doesn't
   match what the variant was authored against, you'll now apply the variant to
   an unintended model. Probably fine in practice (variants are model-agnostic
   prompt overlays in opencode's design), but the previous bug-mode at least
   failed safe by ignoring the variant.

Neither is blocking — both are pre-existing design choices the fix happens to
expose more often. The right place to surface them is a follow-up doc note on
agent.variant semantics.

## Verdict

**merge-after-nits** — add a one-line test in `agent.test.ts` (or wherever the
`session/prompt.ts` chain is exercised) covering the
`(ag.model = undefined, ag.variant = "X", input.variant = undefined)` combo and
asserting `provider.getModel` is invoked with the resolved model. Without a test
this exact regression will recur on the next refactor of the variant chain. Also
update the agent config docs to clarify that `variant` is applied whenever the
agent doesn't pin a contradicting `model`.
