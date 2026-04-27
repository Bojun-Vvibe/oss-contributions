# sst/opencode #24666 — feat(plugin): add model.before hook

- URL: https://github.com/sst/opencode/pull/24666
- Head SHA: `e80afb350650627a14b43391835b8238970fae27`
- Diff: +43/-0 across `packages/opencode/src/session/llm.ts` (+26) and `packages/plugin/src/index.ts` (+17).

## Context / problem

There is no plugin hook fired between "model selected" and "request dispatched", so plugins can't implement hybrid local/cloud routing, cascade fallbacks, or A/B testing without putting a proxy in front of the agent. The existing `chat.params` / `chat.headers` hooks fire too late — by then the provider/model is already locked.

## What the fix does

Defines a new `"model.before"` Hook in `packages/plugin/src/index.ts:240-259` with shape `(input: { sessionID, agent, model, message }, output: { providerID: string, modelID: string }) => Promise<void>`, mirroring `chat.params`'s `(input, output)` convention so the runtime can pre-fill `output` with the original IDs and treat plugin-non-mutation as a no-op.

Wires the dispatch in `packages/opencode/src/session/llm.ts:86-108`: between the existing `l.info("stream", …)` log line and the `Provider.getLanguage`/`Provider.getProvider` resolution, calls `plugin.trigger("model.before", { sessionID, agent: input.agent.name, model: input.model, message: input.user }, { providerID: input.model.providerID, modelID: input.model.id })`. If `rewrite.providerID !== input.model.providerID || rewrite.modelID !== input.model.id`, re-resolves via `provider.getModel(rewrite.providerID as any, rewrite.modelID as any)`, logs `model.before rewrite { from, to }`, and reassigns `input.model = replaced` so all downstream logic (system prompts, params hook, headers hook, `streamText`) sees the new model.

## Specific references

- `packages/plugin/src/index.ts:240-259` — the public Hook contract. JSDoc is solid: explains the no-op semantics, the resolution path, and that an unresolvable pair surfaces as a stream error.
- `packages/opencode/src/session/llm.ts:86-108` — dispatch site. The `if (rewrite.providerID !== ... || rewrite.modelID !== ...)` predicate is the right "plugin didn't change anything" short-circuit.
- `llm.ts:104` — `provider.getModel(rewrite.providerID as any, rewrite.modelID as any)`. The author flags this in the PR body: `getModel` wants branded `ProviderID`/`ModelID` and the plugin output is plain `string`. The `as any` should be replaced with a proper brand-cast (e.g. `ProviderID.make(rewrite.providerID)`) — that's how the codebase brands strings elsewhere.
- `llm.ts:107` — `input.model = replaced`. Mutating the function input is unidiomatic for the rest of `LLM.run`; safer to introduce a local `const model = replaced ?? input.model` and thread that through, but the current approach works because `input` is only consumed within this function.

## Risks / nits

- No regression test. A capability-flag-style hook addition with no test means the wire-up could silently break in a future refactor of `plugin.trigger`.
- `as any` cast on the brand types — should use `ProviderID.make()` / `ModelID.make()` (or whatever the codebase's branded-cast helper is).
- The hook fires unconditionally on every chat completion, even when no plugin registers `model.before`. `plugin.trigger` should be cheap on the empty-handler path; worth confirming.
- "model.before" naming is consistent with existing `chat.before`-style hooks if any; otherwise consider `chat.modelSelect` for clarity. Not blocking.

## Verdict

`merge-after-nits` — the design is right (mirrors `chat.params`, narrow surface, opt-in default-passthrough), but the brand-cast `as any` and missing test should be resolved before merge.
