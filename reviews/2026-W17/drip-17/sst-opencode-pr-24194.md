# sst/opencode PR #24194 — restrict amazon-bedrock provider to curated model allowlist

- **Author:** lgarceau768
- **Head SHA:** 43aa71510cefd94270469294cccc1baac952a360
- **Files:** `packages/opencode/src/provider/provider.ts` (+48 / −0)
- **Verdict:** `request-changes`

## What the diff does

Inside the per-provider model filtering loop in `provider.ts` around
the new lines 1330–1378, the PR introduces a `BEDROCK_ALLOWED_MODELS`
`Set<string>` listing ~30 curated Bedrock model IDs (Claude, Gemma,
Qwen, Mistral, Nemotron, etc.). At line 1394–1395 a new branch deletes
any Bedrock model whose ID is not in that set:

```ts
if (providerID === ProviderID.amazonBedrock && !BEDROCK_ALLOWED_MODELS.has(modelID))
  delete provider.models[modelID]
```

The allowlist is applied unconditionally — there is no flag, env
override, or per-config opt-out.

## Review notes

- **Hardcoded policy in core code.** The right place for this kind of
  curation is `models.dev` / the upstream model registry, not a
  hand-maintained `Set` literal in `provider.ts`. The list will rot
  the moment a new Bedrock model ships and someone forgets to add it
  here. Other providers solve this via the existing
  `configProvider.whitelist` / `blacklist` mechanism a few lines
  below — this PR effectively hardcodes a global whitelist that the
  user cannot override.
- **Breaks legitimate use cases silently.** Bedrock `application
  inference profile` ARNs (the exact scenario PR #24193 documents) are
  used as the model `id`. Those ARNs will never be in
  `BEDROCK_ALLOWED_MODELS`, so users following the official docs will
  see their custom models silently disappear. There is no warning
  log.
- **No test coverage.** A 30-entry constant being load-bearing for
  what users see deserves at least one test asserting that the
  whitelist takes effect and that user-defined custom models survive
  it (which they currently do not).
- **Spelling drift in IDs.** `us.anthropic.claude-opus-4-6-v1` vs
  `us.anthropic.claude-haiku-4-5-20251001-v1:0` — the suffix
  conventions are inconsistent and at least one entry
  (`us.anthropic.claude-sonnet-4-6` with no `-v1` suffix) won't match
  what Bedrock actually returns. This is the failure mode hardcoded
  lists always hit.
- **Motivation isn't in the PR.** If the goal is "hide noisy Bedrock
  catalog entries by default", that should be a tri-state config
  (`bedrock.modelFilter: "all" | "curated" | "custom"`) defaulting
  to `curated`, with `all` preserving today's behavior.

Recommend reworking as: (a) data-driven via models.dev metadata, or
(b) opt-in flag with sane default, plus tests proving custom models
and ARNs survive. As-is this is a regression for any user who has
configured their own Bedrock model entries.

## What I learned

Provider model filters that live next to `whitelist` / `blacklist`
need to compose with them, not preempt them — otherwise the
"escape hatch" the user thinks they have (set
`provider.amazon-bedrock.whitelist`) gets shadowed by an earlier
hardcoded delete. The order of `delete provider.models[modelID]`
checks in this loop is itself a small policy: built-in filters that
run before user filters become un-overridable.
