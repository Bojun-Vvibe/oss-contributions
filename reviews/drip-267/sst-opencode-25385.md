# sst/opencode #25385 — feat(provider): repair malformed SSE JSON via jsonrepair

- **Head SHA:** `d2c17f59cd78ca9eb2cdadfd5d40b2b27842dad8`
- **Files:** `bun.lock`, `packages/opencode/package.json`, `packages/opencode/src/config/config.ts`, `packages/opencode/src/provider/provider.ts`, `packages/opencode/src/provider/sse-repair.ts` (+101), `packages/opencode/test/provider/repair-sse.test.ts` (+107)
- **Verdict:** `merge-after-nits`

## Rationale

Targeted fix for upstream providers (the patch cites Z.AI GLM-5.1) that occasionally emit unescaped quotes inside SSE `data:` payloads, causing the AI SDK's `parseJsonEventStream` to tear down the entire stream. The design is conservative: gated behind `experimental.enable_sse_json_repair` (config.ts:243+), default off, and `repairSSEEvent` in `sse-repair.ts` takes the fast path of `JSON.parse` first and only invokes `jsonrepair` on failure — so valid streams stay byte-identical and pay zero cost. Wiring in `provider.ts:1411` threads `sdkOpts: { repairSSE: boolean }` through `resolveSDK` and applies `repairSSE(res)` before `wrapSSE`, preserving the existing chunk-timeout semantics.

Nits worth addressing before merge: (1) `repairSSEEvent` only inspects lines starting with `data:` — fine for OpenAI-style streams but worth a comment noting that providers using `event:` framing aren't covered; (2) the leading-whitespace preservation logic at sse-repair.ts:~33 is clever but undocumented — inline a one-liner explaining why byte-compat matters for downstream framing; (3) consider logging at `debug` not `info` when repair triggers, otherwise heavy GLM users will see noisy logs.

Risk is low because the feature is opt-in, the failure mode (return original payload) is identical to current behavior, and there's a 107-line test file exercising the repair path.

