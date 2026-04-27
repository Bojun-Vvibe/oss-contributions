# QwenLM/qwen-code #3677 — fix(openai): parse MiniMax thinking tags

- **Repo**: QwenLM/qwen-code
- **PR**: #3677
- **Author**: shenyankm
- **Head SHA**: 77c53e05a2f5d11eff5e08884ba88d6df782130a
- **Size**: +667 / −7 across 13 files; load-bearing edits at
  `core/openaiContentGenerator/converter.ts` (+30 / −5),
  `core/openaiContentGenerator/taggedThinkingParser.ts` (+80 new),
  `core/openaiContentGenerator/provider/minimax.ts` (+26 new),
  `cli/src/ui/hooks/useGeminiStream.ts` (+18 / −2).
- **Closes**: #3669, #3228, #3387.

## What it changes

Three coordinated additions across the OpenAI-compatible response pipeline
to fix MiniMax's quirk of wrapping reasoning content in inline `<think>` /
`<thinking>` tags inside the regular `content` field (not in
`reasoning_content` like other providers):

1. **New `MiniMaxOpenAICompatibleProvider`** at
   `core/openaiContentGenerator/provider/minimax.ts:1-26` with a static
   `isMiniMaxProvider(config)` matcher that parses `config.baseUrl` via
   `new URL(...)` and matches `*.minimaxi.com` only (rejects unparseable
   URLs and unrelated hosts in a try/catch). The provider's
   `getResponseParsingOptions()` returns
   `{ taggedThinkingTags: true }` — the only opt-in surface, so unrelated
   OpenAI-compatible providers stay on the existing default-no-parsing path.
2. **New `parseTaggedThinkingText` + stateful `TaggedThinkingParser`** at
   `core/openaiContentGenerator/taggedThinkingParser.ts:1-80` that splits
   text into Gemini `Part[]` with `thought: true` for `<think>` /
   `<thinking>` content. The streaming version is stateful so a tag split
   across chunks (e.g. `<thi` + `nk>`) is buffered correctly — the test at
   `useGeminiStream.test.tsx:1208-1278` exercises this with a chunked
   async generator.
3. **Converter wiring** at `core/openaiContentGenerator/converter.ts:803-984`:
   non-streaming `convertOpenAIResponseToGemini` calls
   `convertOpenAITextToParts(text, requestContext)` (final = true);
   streaming `convertOpenAIChunkToGemini` calls it with
   `final = Boolean(choice.finish_reason)` and additionally calls
   `convertOpenAITextToParts('', requestContext, true)` on the
   finish-reason chunk to flush any buffered tail. Pipeline at
   `pipeline.ts:520-535` constructs a `TaggedThinkingParser` only when
   `isStreaming && responseParsingOptions?.taggedThinkingTags`, attaches
   it to the `RequestContext`, and the converter dispatches between
   parser-stateful (streaming) and parser-stateless
   (`parseTaggedThinkingText`) at `converter.ts:810-816`.
4. **Bonus CLI fix** at `cli/src/ui/hooks/useGeminiStream.ts:108-110,766-787,852-872`:
   adds a `stripLeadingBlankLines` helper and an `if
   (newGeminiMessageBuffer.trim().length === 0) return newGeminiMessageBuffer;`
   short-circuit at the top of both the content-merge and thought-merge
   branches. Pre-fix, MiniMax streams that begin with `\n\n` chunks were
   committing a `setPendingHistoryItem({ type: 'gemini', text: '' })` too
   early — the empty-text item was rendering as a stray `✦` row above the
   actual response. The two new tests at `:1207-1399` pin both content
   and thought variants of the bug.

## Strengths

- **Right level of gating.** Tagged-thinking parsing is opt-in via
  `getResponseParsingOptions()` per provider, defaulting to the existing
  no-parsing path for everything that isn't MiniMax. The risk of
  regressing unrelated providers (e.g. an OpenAI-compatible chat that
  legitimately emits `<think>` as user-facing prose) is structurally zero.
- **The `*.minimaxi.com` matcher is correctly bounded** at `minimax.ts`'s
  `isMiniMaxProvider` (verified by the three test cases at
  `minimax.test.ts:27-58`: official `api.minimaxi.com` matches, subdomain
  `gateway.minimaxi.com` matches, `api.openai.com` and unparseable URL and
  missing-baseUrl all reject). The try/catch around `new URL(...)` is the
  right defense against mistyped configs.
- **Stateful streaming parser is necessary, not optional.** Tags split
  across chunks (the `<thi` + `nk>` case explicitly tested at
  `useGeminiStream.test.tsx:1234`) cannot be handled by a stateless
  per-chunk parser, and the converter correctly distinguishes the
  per-call instance (attached to `requestContext.taggedThinkingParser`)
  from the per-string `parseTaggedThinkingText` fallback for non-streaming.
- **The finish-reason flush at `converter.ts:973-975`** —
  `else if (choice.finish_reason) { parts.push(...convertOpenAITextToParts('', requestContext, true)); }`
  — is the right way to drain buffered tail content when the last chunk
  has no `delta.content`. Without this, a stream that ends mid-tag would
  silently drop the trailing thought.
- **The CLI blank-chunk bonus fix is genuinely orthogonal but tightly
  related.** MiniMax surfaces this bug because it tends to emit
  whitespace before content; the same code path would misbehave for any
  provider that does the same. The fix is correctly local
  (`useGeminiStream.ts`-only) and pinned by two tests that name the
  failure mode (`does not render leading blank content chunks as an
  empty assistant item`, `does not render leading blank thought chunks
  as an empty thought item`).
- **Test coverage is genuinely substantial.** +147 lines in
  `useGeminiStream.test.tsx`, +80 in `minimax.test.ts`, +251 in
  `converter.test.ts` exercising tagged thinking parse paths. The
  fake-timer + chunked-async-generator pattern is the right shape for
  testing throttle-gated streaming UI state.

## Concerns / nits

- **Provider detection is host-suffix-only.** `isMiniMaxProvider` matches
  `*.minimaxi.com` but says nothing about the model field. A user pointing
  a `gpt-4o`-like model at `api.minimaxi.com` (which would be
  unusual but legal — proxy setups exist) would get tagged thinking
  parsing applied to a response that may legitimately contain the literal
  string `<think>`. The risk is small but real; a model-name guard
  (`config.model.toLowerCase().includes('minimax')`) as a second
  conjunct would tighten the match. Worth at least a doc comment on
  the matcher explaining the assumption.
- **`stripLeadingBlankLines` regex `/^(?:[ \t]*\r?\n)+/`** at
  `useGeminiStream.ts:108-110` strips runs of "indent-then-newline"
  patterns. That's the right behavior for typical MiniMax `\n\n`
  preambles, but it does NOT strip a trailing space-only line
  (e.g. `"   "` with no newline) — if a future provider's preamble
  ends with whitespace-only-no-newline, the empty-row bug returns. A
  more permissive `/^\s+/` would catch that too, at the cost of
  trimming legitimate-but-unusual leading whitespace from the first
  visible chunk. The PR's tighter regex is the safer choice; just
  worth a comment naming the tradeoff.
- **The `setPendingHistoryItem({ type: 'gemini', text: '' })` flow** at
  `useGeminiStream.ts:780` still does the `addItem(pendingHistoryItemRef.current, ...)`
  finalize-and-replace even after the early return for blank chunks.
  That means in the all-blank scenario, no pending item is ever
  created — which is correct — but if the stream then emits a
  non-blank chunk much later, the path through line 780-788 takes
  the new-buffer-with-stripped-leading-blanks. Worth a one-line
  comment at the early-return explaining "pending item creation
  is deferred until first non-blank chunk".
- **`parseTaggedThinkingText` (the non-streaming entry point) at
  `taggedThinkingParser.ts`** isn't visible in the diff snippet I
  inspected, but the converter at `converter.ts:813-815` falls back
  to it whenever `requestContext.taggedThinkingParser` is unset
  (i.e. non-streaming responses). Worth confirming the stateless
  parser produces identical outputs to the stateful one when fed
  the same complete string — a `parser.parse(s, true)` vs.
  `parseTaggedThinkingText(s)` round-trip pin test would catch
  any future drift between the two implementations.
- **No test for the finish-reason-empty-content flush** at
  `converter.ts:973-975`. The branch is exercised implicitly by some
  of the chunked tests, but a focused pin (`given a stream that
  emits "answer<thi" then "nk>still" then "" with finish_reason="stop",
  the converted parts include `{text: "answer "}` and
  `{text: "still", thought: true}`) would lock the contract against
  future "simplification" PRs that drop the finish-reason flush.
- **The CLI test imports `findLastSafeSplitPoint` from
  `markdownUtilities.js`** at `useGeminiStream.test.tsx:40` and mocks
  it via `vi.mocked(findLastSafeSplitPoint).mockImplementation(...)`.
  The `vi.mock` declaration for that module isn't visible in the
  diff hunk; if it's at the top of the file outside the diff, fine
  — if not, the test will throw at module load time. Worth a quick
  cross-check.

## Verdict

**merge-after-nits.** Right architectural shape: provider-gated opt-in
`responseParsingOptions`, stateful streaming parser threaded through
`RequestContext`, finish-reason flush on the streaming path, and a
genuinely-orthogonal-but-justified CLI blank-chunk fix bundled in. Test
coverage is substantial across all three phases. Before merge: (1) add
a model-name conjunct to `isMiniMaxProvider` or document the host-only
assumption, (2) add a stateful-vs-stateless parser parity pin test,
(3) add a focused finish-reason-empty-content flush test, and (4) add
a one-line tradeoff comment to `stripLeadingBlankLines`'s regex.
