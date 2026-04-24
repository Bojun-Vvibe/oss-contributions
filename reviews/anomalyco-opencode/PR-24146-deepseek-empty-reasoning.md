# PR #24146 — preserve empty `reasoning_content` for DeepSeek V4 thinking mode

**Repo:** sst/opencode • **Author:** heimoshuiyu • **Status:** open
• **Net:** +10 / −15

## Change

Drop the `if (reasoningText)` truthy guard in
`provider/transform.ts` `normalizeMessages` so the assistant message
always carries `providerOptions.openaiCompatible[field]` — even when
the reasoning text is `""`. DeepSeek V4 thinking mode emits empty
`reasoning_content` on assistant turns inside multi-step tool chains,
and the upstream API rejects subsequent turns when that field is
missing.

## Load-bearing risk

The fix relies on a downstream behavior in `@ai-sdk/openai-compatible`:
the request builder spreads `...metadata` into the outgoing message,
which carries `reasoning_content: ""` even though that package's own
parser would discard it on the response side. So this PR is correct
only because the *outbound* path keeps empty strings while the
*inbound* path drops them. If a future AI-SDK refactor harmonizes the
two (likely — the linked vercel/ai#13203 is doing exactly that for
`@ai-sdk/deepseek`), `[field]` will quietly become `undefined` again
and the bug returns silently. There is no opencode-side test that
asserts the wire payload contains `reasoning_content: ""`.

A second sharp edge: the field is now set unconditionally for **every**
assistant message, including providers that don't understand
`reasoning_content` at all. Most OpenAI-compatible servers tolerate
unknown keys, but some strict gateways (Azure inference SDK, some
enterprise proxies) reject unknown message fields with a 400. The
old guard incidentally protected non-thinking providers; removing it
widens the blast radius beyond DeepSeek.

## Concrete next step

Add a regression test at the transform boundary that snapshots the
outgoing payload for three cases: (1) DeepSeek with non-empty
reasoning, (2) DeepSeek with empty-string reasoning, (3) a vanilla
OpenAI-compatible provider (no thinking). Case (2) locks the contract
this PR depends on; case (3) catches the day a strict downstream
provider starts rejecting the now-always-present field. Pair this
with a one-line guard at message-build time that only attaches
`[field]` when the provider's capability map advertises thinking
support, instead of trusting every OpenAI-compatible provider to
ignore it.

## Verdict

Correct fix for the reported symptom, fragile due to coupling with
upstream AI-SDK behavior. Worth landing with a snapshot test.

## What I learned

Empty string ≠ absent: when a field's *presence* is part of the
contract, truthy guards in serialization are landmines. This is the
same shape as the W17 drip-5 User-Agent header collision — both are
"silent default overrode the user's explicit value (or its absence)."
