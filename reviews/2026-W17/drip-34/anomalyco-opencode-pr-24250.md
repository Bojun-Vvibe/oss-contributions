# anomalyco/opencode #24250 — Complete DeepSeek reasoning_content round-trip for multi-turn conversations

- **Repo**: anomalyco/opencode
- **PR**: [#24250](https://github.com/anomalyco/opencode/pull/24250)
- **Head SHA**: `b6a7cdb43686f8aadbfec6fff7b80d2f5aedd10a`
- **Author**: knefenk
- **State**: OPEN
- **Verdict**: `merge-after-nits`

## Context

DeepSeek's chat API requires every assistant turn to carry a
`reasoning_content` field once thinking mode is enabled, otherwise the
provider rejects the multi-turn payload. The existing transform layer in
`packages/opencode/src/provider/transform.ts` only injected
`reasoning_content` for messages that already had a `reasoning` part —
which means historical assistant messages persisted to the DB **before**
reasoning mode was enabled, and DeepSeek models routed via OpenRouter
(API id doesn't include `deepseek` literal), would both fail re-hydration
mid-conversation. This PR closes both gaps.

## Design

Three changes, each correct:

1. **Provider routing** — `provider.ts:1180` now defaults
   `interleaved` to `{ field: "reasoning_content" }` for any model that
   advertises `model.reasoning`. Previously `interleaved` defaulted to
   `false` unless the catalog entry explicitly set it, so a generic
   reasoning-capable model never opted into the field-injection path. The
   `?? existingModel?.capabilities.interleaved ?? (...)` chain preserves
   user overrides.

2. **OpenRouter detection** — `transform.ts:179` widens the DeepSeek
   sniff from `model.api.id.includes("deepseek")` to also check
   `model.id.includes("deepseek")`. This is the actual reported
   regression: when the user routes `deepseek/deepseek-r1` through
   OpenRouter, `model.api.id` is `openrouter` and the old check missed it.

3. **Backfill empty reasoning on historical turns** —
   `transform.ts:228-256` adds an unconditional second pass: if
   `model.capabilities.reasoning`, every assistant message gets
   `reasoning_content: ""` injected via `providerOptions[sdk]`, with a
   string-content fallback that appends an empty `reasoning` part. Comment
   on lines 226-228 correctly documents the "DB-loaded historical turn"
   case.

The `sdk = sdkKey(model.api.npm) ?? "openaiCompatible"` lookup at line
198 is the right hardening — the previous code hardcoded
`openaiCompatible`, which would silently no-op for any provider whose
SDK key resolves differently (vercel `@ai-sdk/deepseek`, etc.).

## Risks / Nits

1. **Two passes both touch every assistant message.** When `interleaved`
   is configured AND `reasoning` is enabled (the common DeepSeek path),
   the first block at line 197-224 runs, then the second block at
   line 230-256 runs again and overwrites `reasoning_content` with `""`
   for any message that didn't have a reasoning part to extract. Verify
   this is intended: the alternative is to merge the two passes so the
   empty default fills only when the first pass found no reasoning to
   carry forward. As written, an assistant turn whose reasoning got
   stripped (e.g., compaction) gets `""` which is what DeepSeek wants —
   so it's probably correct, but a 1-line comment on the second block
   explaining "this intentionally overrides the first block for messages
   with no reasoning parts" would prevent future revert.

2. **`{ type: "reasoning" as const, text: "" }`** at line 252 will, on
   the next turn, hit the first block's `filter(part => part.type ===
   "reasoning")` extraction at line 203 and produce an empty
   `reasoningText`. Net effect identical, but it does mean the message
   now has a synthetic reasoning part stored in the canonical content
   array — make sure the rollout/persistence layer doesn't echo this
   back as a visible reasoning UI block to the user.

3. No new test was added. Given this is a known multi-turn regression,
   a snapshot test in the provider/transform suite asserting that an
   assistant message with no `reasoning` part gets `reasoning_content:
   ""` injected for `deepseek*` models would lock both regressions
   (historical-turn + OpenRouter routing).

## What I learned

The fix is interesting because it shows a class of bug: provider-specific
wire-format invariants ("every assistant turn must carry field X") that
hold at write time but are violated by **stored history** when the user
toggles a capability mid-conversation, or by **alternative routing** when
the provider is reached via an aggregator. The systemic fix is the
unconditional pass that re-establishes the invariant at request-build
time, not at write time — because by then it's too late.
