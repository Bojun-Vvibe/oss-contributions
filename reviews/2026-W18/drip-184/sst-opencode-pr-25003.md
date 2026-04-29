---
pr: sst/opencode#25003
sha: 48ac948493bf6b35de93bf873c205a7492ac4c06
verdict: merge-after-nits
reviewed_at: 2026-04-30T00:00:00Z
---

# fix: preserve Anthropic thinking block signatures when switching Claude models

URL: https://github.com/sst/opencode/pull/25003
Files: `packages/opencode/src/provider/transform.ts`, `packages/opencode/src/session/message-v2.ts`
Diff: 51+/1-

## Context

The Anthropic API requires every previously-emitted reasoning/`thinking`
block to be echoed back on the next turn with its original cryptographic
`signature` field intact, otherwise it 400s with
`messages.N.content.0.thinking.signature: Field required`. The existing
session→model-message converter at `message-v2.ts:838-963` was using a
single boolean `differentModel` to decide whether to include
`providerMetadata` on emitted `reasoning` parts: any model switch (even
`claude-sonnet-4.5` → `claude-opus-4.7`) wiped the signature, and the
very next request from history got rejected. The bug rate scales with
how often the user does the textbook "switch to a stronger Claude for
the hard part" workflow.

## What's good

- The fix at `message-v2.ts:840-855` introduces a narrower predicate
  `keepReasoningMetadata` that is `!differentModel || (isAnthropicFamily(new) &&
  isAnthropicFamily(old))`, codifying the actual contract: the signature
  is portable across Anthropic models (Sonnet ↔ Opus ↔ Haiku, direct API
  ↔ Bedrock) but must be dropped when crossing into a non-Anthropic
  provider that can't validate it. Comment at `:843-849` explicitly cites
  the API error string, which is exactly the breadcrumb a future debugger
  needs.
- `isAnthropicFamily` at `:852-855` recognises three paths — `providerID
  === "anthropic"`, `providerID === "amazon-bedrock"`, and the safety net
  `modelID.includes("claude")` — covering Bedrock's
  `amazon-bedrock/anthropic.claude-*` ids and any custom provider config
  that routes Claude through a non-canonical providerID name.
- The downstream emit at `:960-967` switches from `differentModel ? {} :
  { providerMetadata: ... }` to `keepReasoningMetadata ? {
  providerMetadata: ... } : {}` — minimal mechanical change at the actual
  emission site, with the comment again explaining why this isn't just
  the obvious `differentModel` ternary.
- The companion defence in `transform.ts:96-122` is the load-bearing
  belt-and-braces piece: even when upstream metadata IS lost (replayed
  session, partial migration, third-party imports), the Anthropic
  request is now sanitised at the provider edge by dropping any
  `reasoning` part whose `providerMetadata.anthropic.signature` is
  missing/empty/non-string. The empty-content guard at `:115-117`
  correctly preserves message structure (Anthropic also rejects empty
  content arrays) by substituting a single empty text part — the
  conservative choice over deleting the whole message.
- Gate at `transform.ts:101` keys on `model.api.npm` rather than
  `model.providerID` so it fires for both `@ai-sdk/anthropic` and
  `@ai-sdk/amazon-bedrock` without depending on user provider naming.

## Nits

- The signature-shape check at `transform.ts:106` (`sig && typeof sig
  === "string" && sig.length > 0`) and the modelID-includes check at
  `message-v2.ts:855` are duplicating the implicit "what is an Anthropic
  thinking block" predicate in two places — extract a shared
  `hasValidAnthropicSignature(part)` helper colocated with the
  `isAnthropicFamily` predicate so the next provider that grows a
  signed-reasoning concept can compose with it.
- The `(part as any).providerMetadata` cast at `transform.ts:106` works
  but loses the type system's help — if `ReasoningPart` already has a
  typed `providerMetadata.anthropic.signature` field, prefer the typed
  access; if not, add a small typed accessor in `message-v2.ts` next to
  where the part is constructed and reuse it here.
- No test in the diff covering the Sonnet→Opus replay scenario or the
  defensive sanitiser at `transform.ts:96-122`. Two integration tests
  would lock the contract: (a) build a 3-message history with a
  signature-bearing reasoning part, swap providerID/modelID between
  two Anthropic models, run through `toModelMessages` + `transform`,
  assert `providerMetadata.anthropic.signature` survives end-to-end;
  (b) construct a history part with `providerMetadata` deliberately
  set to `{}`, assert the `transform` filter drops it and substitutes
  empty-text for the message.
- The `modelID.includes("claude")` fallback in `isAnthropicFamily` at
  `:855` is correct for today but will accept hypothetical
  `claude-disguise-v1` from a non-Anthropic provider — fine because the
  request will simply be rejected by that provider's validator, but
  worth a comment that this is an over-permissive heuristic chosen to
  avoid false negatives on signature preservation.
- One-line CHANGELOG / release-notes entry would help operators on
  Bedrock who've been working around this with `compact` — they need to
  know they can stop.

## Verdict reasoning

Correct two-layer fix (message-builder predicate narrowing + provider-edge
sanitiser) for a real, frequently-hit Anthropic API rejection. The
predicate captures the actual contract (signature is Anthropic-family
portable) rather than the over-conservative "any model change wipes
signature" of the prior code. Defensive transform-layer scrub means
older sessions and migrations also get healed instead of needing manual
intervention. Nits are about extracting shared helpers, typed access,
and adding regression tests — none blocking.
