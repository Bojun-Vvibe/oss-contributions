# sst/opencode #24666 — feat(plugin): add model.before hook

- URL: https://github.com/sst/opencode/pull/24666
- Head SHA: `e80afb350650627a14b43391835b8238970fae27`
- Verdict: **merge-after-nits**

## Review

- New plugin hook fires inside the LLM dispatch path at `packages/opencode/src/session/llm.ts:86-108`, with the output object pre-filled with `{providerID, modelID}` so a plugin that doesn't touch the output is a no-op. Right shape for a routing hook — input carries `sessionID/agent/model/message` so plugins have enough signal to make A/B / cascade-fallback / hybrid-routing decisions, and the change-detection at `:103` (`rewrite.providerID !== input.model.providerID || rewrite.modelID !== input.model.id`) avoids re-resolving the model when the plugin no-ops.
- `provider.getModel(rewrite.providerID as any, rewrite.modelID as any)` at `:104` uses two `as any` casts to bypass the strongly-typed provider/model registry — these will silently swallow an invalid pair until `getModel` itself rejects. That's fine for runtime behavior (an unresolvable pair surfaces as an error, as the docstring promises) but the casts should at least be commented "plugin-supplied IDs are runtime-validated by getModel" so a future cleanup PR doesn't try to tighten them and break the hook.
- `input.model = replaced` at `:108` mutates the input object in place. The function takes `input` and threads it through subsequent `Effect.all([...])` consumers; mutation is OK only because no other reader has captured the old reference yet, but if a future refactor stashes `input.model` earlier in the function (e.g. for telemetry) the rewrite will silently desync. A non-mutating `const model = rewrite.providerID !== ... ? replaced : input.model` plus passing `model` explicitly to downstream calls would be safer; lower priority.
- Plugin contract at `packages/plugin/src/index.ts:243-259` is well-documented (use cases enumerated, no-op behavior named, error semantics named). One missing piece: no example or doc note about *ordering* when multiple plugins register `model.before` — `plugin.trigger` semantics need to be either "last-write-wins" or "fold left" and the chosen semantic should be in the docstring. Without that, two plugins that both want to route will silently conflict.
