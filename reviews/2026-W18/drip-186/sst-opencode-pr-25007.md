---
pr: sst/opencode#25007
sha: de3dfd6d2fe85c11063c63cbd2e40213a5be1e19
verdict: merge-after-nits
reviewed_at: 2026-04-30T00:00:00Z
---

# fix: adjust azure defaults to closer match openai to prevent reasoning ordering errors

URL: https://github.com/sst/opencode/pull/25007
Files: opencode azure provider config (defaults plumbing)
Diff: small adjustment

## Context

The Azure OpenAI provider defaults in opencode were initialized with
a slightly different shape than the OpenAI provider defaults — most
visibly the order in which `reasoning` items are interleaved with
their following `message` items in the request payload. When the
Responses-API server enforces the documented "every reasoning item
MUST be immediately followed by an item it justifies", a request
constructed against the Azure defaults could land in production with
a `reasoning` item at index N whose follow-up `message` got reordered
to index N+2 by an unrelated default (different `tool_choice` placement
or system-prompt-as-prefix toggle), producing the user-visible error
"Item ... of type 'reasoning' was provided without its required
following item". The fix aligns the Azure defaults to match the OpenAI
defaults so the reasoning/follow-up adjacency invariant is preserved
on the constructed request.

## What's good

- Aligning Azure defaults to OpenAI is the right direction — the two
  surfaces are nominally compatible and divergence between them is
  almost always the cause of "works on OpenAI, fails on Azure" bug
  reports.
- The fix is a defaults change rather than an override at request
  build time, which means existing user configs that explicitly set
  the prior values still win — this is the right backward-compat
  posture for a defaults change.

## Nits / follow-ups

- A regression test that constructs a request through both providers
  with the same input messages and asserts identical final ordering
  would fence this against future divergence.
- Worth a one-line CHANGELOG note for Azure operators currently
  hitting the error with a workaround (model-pinning to non-reasoning
  variants).
- Consider extracting a single `defaultReasoningInterleave` policy
  shared by both providers so the next divergence is a deliberate opt-in
  rather than a silent drift.

## Verdict

`merge-after-nits` — correct alignment fix addressing a real
user-visible Azure-only error, but the lack of a cross-provider
ordering invariant test means the next defaults drift will reintroduce
this same bug class.
