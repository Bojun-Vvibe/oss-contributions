# sst/opencode PR #25861 — fix(provider): preserve anthropic provider-executed tool pairs

- Head SHA: `5c1c3b74b1159c62c10c52c6d3be59b6f7e11163`
- Author: `nexxeln`
- Size: +126 / -8
- Verdict: **merge-as-is**

## Summary

Refines the existing anthropic-only `[tool_use, …, text] → split` normalizer
in `packages/opencode/src/provider/transform.ts:124-153` so that
provider-executed tool calls (e.g. anthropic `code_execution`,
`web_search` server-side tools) are not torn apart from their inline
`tool-result` and any trailing assistant text. Previously the splitter
classified *any* `tool-call` as "must move to second message", which for
provider-executed tools produced a payload that the AI SDK rejected
because the SDK intentionally co-locates server tool-call + tool-result
+ commentary in a single assistant turn.

## What the diff actually does

1. `transform.ts:133-141` — builds a `Set<toolCallId>` of
   `providerExecuted === true` tool-call ids, then defines
   `isClientToolPart(part)` returning true only for non-provider-executed
   `tool-call` parts and `tool-result` parts whose id is *not* in the
   provider-executed set. Provider-executed `tool-result` parts are
   therefore explicitly classified as "non-client" (i.e. they stay with
   the trailing text in the first message).
2. `transform.ts:142` — the `first` index now searches for the first
   *client* tool-call (`part.type === "tool-call" && part.providerExecuted !== true`).
   If none exists, the message is returned unchanged — meaning a turn
   that contains only provider-executed tool work + text is now a
   no-op for the splitter, which is the correct AI SDK contract.
3. `transform.ts:143-149` — split predicate now uses
   `isClientToolPart` everywhere so the produced shape is
   `[non-client parts (text + provider-executed pair)…] then [client tool-calls…]`.
4. The doc-comment update at lines 122-133 cleanly marks the
   "provider-executed tools are different" rationale and removes the
   obsolete "root cause unknown upstream" paragraph that no longer
   applies for the provider-executed case.

## Test coverage

Three new tests at `packages/opencode/test/provider/transform.test.ts:1501-1611`
pin the three behavioral regions:

- Pure client `[tool-call, tool-result, text]` — splits as before
  (text first, tool pair second).
- Pure provider-executed `[tool-call(provider), tool-result, text]` —
  passthrough, unchanged (`expect(result).toHaveLength(1)`).
- Mixed `[client tool-call, provider tool-call, provider tool-result, text]` —
  first message contains the provider pair + text, second contains the
  lone client tool-call, exactly the shape the AI SDK accepts.

The third test is the high-value one: it exercises the
`providerExecuted` set construction across multiple parts and asserts
the partition is by id-membership rather than positional, which is the
non-trivial semantic this PR introduces.

## Why merge-as-is

- Targeted bug fix with no behavior change for non-anthropic providers
  (the whole block is gated on `model.api.npm` membership at line 124).
- The classification helper is correct: `tool-result` parts are
  uniquely identified by `toolCallId` so id-set membership is the right
  partition key (no ambiguity, no false positives).
- The `findIndex` at line 142 already filters by
  `providerExecuted !== true`, so a turn that opens with a
  provider-executed tool-call followed by client tool-call is still
  detected (the client one is found, splitter runs).
- Test triplet covers the three meaningful permutations and explicitly
  asserts both message count and content structure; the
  `programmatic-tool-call` input shape and `code_execution_result`
  output shape match the real anthropic provider-executed payload.
- Net diff is +126/-8 with -8 of removed comment + 4 changed lines of
  logic, well within "small fix" surface; no public type changes.

## Optional nits (not blocking)

- `transform.ts:139` — the type annotation
  `(part: (typeof parts)[number])` is correct but slightly awkward;
  if the file already has a named `MessagePart` union it would read
  cleaner. Pure stylistic.
- The new doc-comment at lines 132-133 could mention by name that this
  applies to anthropic `code_execution`, `web_search`, `bash_20250124`
  etc. server tools, so the next reader doesn't have to grep the AI SDK
  source to understand which tools are "provider-executed". Optional.
