# sst/opencode PR #25496 — fix(opencode): refresh provider models and resolve merge fallout

- **Repo:** sst/opencode
- **PR:** #25496
- **Head SHA:** `6258f0726a89dd9bcc9ab831cba4dc4806fe1afd`
- **Author:** jameswright1562
- **Title:** fix(opencode): refresh provider models and resolve merge fallout
- **Diff size:** +492 / -68 across 9 files
- **Drip:** drip-292

## Files changed

- `packages/opencode/src/provider/provider.ts` (+300/-66) — main change: adds `Event.Updated`, `Interface.refresh`, `ProviderScan` machinery, `normalizeScannedModels`, `toScannedModel`, and threads a refresh promise through `State`.
- `packages/opencode/src/cli/cmd/tui/context/sync.tsx` (+17/-0) — adds a `provider.updated` bus handler that re-fetches `config.providers` + `provider.list` and reconciles into the SolidJS store.
- `packages/opencode/src/effect/bootstrap-runtime.ts` (+2/-0) and `packages/opencode/src/project/bootstrap.ts` (+1/-0) — wire `Provider.defaultLayer` into the bootstrap layer chain.
- `packages/opencode/test/provider/provider.test.ts` (+160/-0) — new tests for the refresh flow.
- `packages/app/src/custom-elements.d.ts`, `packages/enterprise/src/custom-elements.d.ts` — convert symlink shims to `/// <reference path=...>` triple-slash directives.
- `packages/sdk/js/src/v2/gen/types.gen.ts` (+9/-0) — generated types delta.

## Specific observations

- `provider.ts:36-44` — `Event.Updated` carries only `providerIDs: string[]`. The TUI consumer at `sync.tsx:362-378` ignores that payload and re-fetches the *full* provider list every time. That's a correctness-safe choice but defeats the purpose of including `providerIDs` — either consume the IDs to do partial reconciliation, or drop the payload to `Schema.Struct({})`.
- `provider.ts:983-989` (`State`) — `refreshing?: Promise<void>` is the in-flight dedupe guard. Good, but no test in the new `provider.test.ts` (lines 1-160) appears to cover concurrent `refresh()` callers awaiting the same promise; recommend a test that fires `refresh()` twice in parallel and asserts the underlying scan runs once.
- `provider.ts:1099-1112` (`normalizeScannedModels`) — uses `ModelID.make(id)` for every key but does no validation that scan-returned IDs are unique across providers; if two scans report the same ID the second silently wins via `Object.fromEntries`. Worth a `log.warn` on collision.
- `provider.ts:1114-1148` (`toScannedModel`) — manual deep clone of `cost`, `cost.cache`, `experimentalOver200K.cache`, `limit`, `options`, `headers`, and `variants`. This is brittle — adding a new nested field to `Model` will silently lose it on round-trip. Consider `structuredClone(model)` then strip non-serializable bits, or codegen this from the schema.
- `sync.tsx:362-378` — `void Promise.all([...])` swallows rejections. If either `config.providers` or `provider.list` throws (network, auth), the store silently keeps the stale list. At minimum chain a `.catch(log.error)`.
- `custom-elements.d.ts` symlink → triple-slash conversion is unrelated to "refresh provider models" and should be split into its own PR; bundling it under "merge fallout" makes the change harder to bisect.
- `bootstrap-runtime.ts:18-19` — adding `Provider.defaultLayer` to `BootstrapLayer` is correct, but the import in `bootstrap.ts` (line 6) appears unused inside this hunk — confirm it's referenced elsewhere in the file or remove.

## Verdict: `request-changes`

The refresh plumbing is the right shape, but three issues block a clean merge: (1) the bus event payload is defined but ignored by the only consumer, (2) the `Promise.all` in `sync.tsx` swallows errors so failures are invisible, and (3) the symlink/triple-slash conversion should be a separate PR. Address these and the rest reads cleanly.
