# sst/opencode PR #24612 — dynamically load model list from LM Studio

- **PR**: https://github.com/sst/opencode/pull/24612
- **Author**: @ricardofiorani
- **Head SHA**: `36157b68d41a7342aa4aab4b13c09523c9a97374`
- **Size**: +136 / −1
- **Files**: `packages/opencode/src/provider/model-cache.ts` (new), `packages/opencode/src/provider/models.ts`

## Summary

Adds a `ModelCache` namespace that, at provider-resolution time, hits a configured LM Studio (or Apertis) `/v1/models` endpoint and merges the returned model list into the static snapshot — so locally-loaded LM Studio models show up without a `models.json` ship.

## Verdict: `merge-after-nits`

The shape is right (lazy-fetched, cached per-providerID, fail-soft to snapshot), but two surfaces are too generous: a synchronous network call sits in `Provider.get()` (every caller pays the latency once at boot), and the per-model defaults are hard-coded with `cost: { input: 0, output: 0 }` plus a 128k context that LM Studio servers do not actually advertise. Both bite operators silently.

## Specific references

- `packages/opencode/src/provider/model-cache.ts:1-86` — the `Bun.fetch(url, { signal: AbortSignal.timeout(10_000) })` is good (it bounds the worst case), but on first call inside `Provider.get()` every caller pays up to 10 s if the user has `lmstudio` enabled but no LM Studio running. The `cache[providerID]` short-circuit at L9 only helps the *second* call. Either wrap the first fetch in a much shorter timeout (1–2 s — LM Studio is loopback by default) or push the whole fetch behind an `await`-once `lazy()` so the second `Provider.get()` doesn't repay the cost.
- `packages/opencode/src/provider/model-cache.ts:53-67` — the synthesized `ModelInfo` literal sets `attachment: true`, `tool_call: true`, `cost: { input: 0, output: 0 }`, and `limit: { context: 128000, output: 4096 }` for every model regardless of what LM Studio reports. A user running a 4k-context Qwen 1.5B locally will get truncation surprises because the runtime trusts `limit.context`. At minimum, default `tool_call: false` (most local quants don't, and the user can override) and either probe `/v1/models/<id>` for the real context or set `limit.context: 8192` as a safe floor.
- `packages/opencode/src/provider/models.ts:148-198` — the `disabled_providers`/`enabled_providers` plumbing duplicates logic that should live next to provider config, and the `lmstudioAllowed` gate hard-codes `"lmstudio"` four times. If a third provider lands using the same OpenAI-compat shape (the diff already adds Apertis to `model-cache.ts` but not to this gate), this branch needs to grow. Extract a `resolveDynamicProviders(config)` helper that returns `Array<{ id, options }>` and loop, so adding Apertis here is a one-row config change instead of a copy-paste.
- `packages/opencode/src/provider/models.ts:177-194` — when LM Studio responds with zero models (server is up but no model loaded), `lmstudioModels` is `{}`, the `Object.keys(...).length > 0` guard short-circuits, and the snapshot fallback applies. Good. But when the snapshot *also* has no entry, `providers["lmstudio"]` stays `undefined` silently — a user who configured LM Studio sees nothing in the picker with no diagnostic. A single `log.warn("lmstudio configured but no models discovered", { baseURL })` at the bottom of the `lmstudioAllowed` block is a 5-second fix.

## Nits

1. Tighten the first-call timeout to ~2 s, or push behind `lazy()` so the second call is free.
2. Drop `tool_call: true` and `cost: 0/0` defaults — log a warning that these need explicit config.
3. Extract a `resolveDynamicProviders(config)` helper and loop over it in `models.ts` instead of branching per-providerID.
4. Warn (not error) when the configured LM Studio endpoint responds with zero models.

## What I learned

The "ship a static `models.json` snapshot, but allow dynamic enrichment for local providers" pattern is exactly the right shape for an agent IDE — the long tail of local-Ollama / local-LM-Studio installs cannot be enumerated at build time. The one trap is that synthesized model metadata silently lies about cost and limits if the user's local server doesn't have an opinion. A future-proof shape would have the dynamic-enrichment helper return `Partial<ModelInfo>` and merge over a `defaults: { ... cost: undefined, limit: undefined }` template that the cost panel knows how to render as "unknown" rather than "$0.00".
