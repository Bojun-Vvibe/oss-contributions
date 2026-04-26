# QwenLM/qwen-code #3617 — fix(core): split tool-result media into follow-up user message for strict OpenAI compat

- **Author:** mohitsoni48
- **Head SHA:** `4f8d5e02b55cf2cf21ad3755dee5587bfac8cfd3`
- **Size:** +41 / -0 (1 file, draft)
- **URL:** https://github.com/QwenLM/qwen-code/pull/3617

## Summary

`createToolMessage` produces `role: "tool"` messages whose `content`
array can carry `image_url`, `input_audio`, `video_url`, or `file`
parts. The OpenAI Chat Completions spec allows only string or text-part
content on tool-role messages, so strict OpenAI-compatible servers
(LM Studio is the primary report; presumably others) reject these
payloads with `400 Invalid 'messages' in payload`. The existing code
in `converter.ts` papers over this with
`as unknown as ChatCompletionContentPartText[]` — a literal "we know
this isn't spec, hope the server accepts it" cast. Many do not.

## Specific findings

- **The fix is the correct shape *if* the project decides to flip the
  default.** `packages/core/src/core/openaiContentGenerator/converter.ts:444-484`
  walks the existing `toolMessage.content`, partitions parts into
  text vs media (`image_url` / `input_audio` / `video_url` / `file`),
  rewrites the tool message to be text-only (with a fallback string
  `"[media attached in following user message]"` so empty-text tool
  responses don't hit the spec's "must be non-empty" path), and then
  appends a synthetic `role: "user"` message carrying the media plus
  a context-bearing `"(attached media from previous tool call)"`
  prefix. Conversation order is preserved, the model still sees the
  media, and the wire payload becomes spec-compliant.

- **The author flagged the real blocker themselves: 11 existing tests
  in `converter.test.ts` will fail.** Per the PR body, tests added
  for #1520 (e.g. `should preserve MCP multi-text + multi-image content
  in tool message`, `should not leak to user message`) explicitly assert
  the *old* behaviour as the contract. This isn't a one-side bug —
  somebody made a deliberate call that some permissive providers *want*
  the media inside the tool message. Flipping the default needs an
  explicit decision, not just a code change.

- **The four proposed paths in the PR body are reasonable; one is
  cleanest.** Path 2 (per-provider `splitToolMedia` flag) keeps the
  blast radius minimal and lets LM Studio users opt in without
  retroactively breaking flows that depended on the old behaviour. The
  existing `isDeepSeekProvider`-style detection pattern (see
  `provider/deepseek.ts:25-35`, hot off PR #3620) shows the project is
  already comfortable with per-provider quirks gates, so this fits
  the codebase. Path 3 (auto-detect on `localhost:1234`) is brittle
  and shouldn't ship.

- **Repro is well-attested.** Issue #3616 has the exact LM Studio +
  Playwright MCP failure trace, and the author confirmed an
  out-of-band patch to bundled `cli.js` resolves the 400 — so the
  diagnosis is sound. Affected tools listed (Playwright screenshot,
  filesystem `read_media_file`, computer-use screenshot, android
  screenshot) are realistic and the workaround (`@path` ingestion)
  genuinely doesn't apply to in-flight browser flows.

- **Minor implementation notes worth folding in if path 1 wins.**
  - The `as unknown as ChatCompletionContentPartText[]` cast on the
    follow-up `user` message (line 483) is the same kind of typing
    bypass the original code was doing — should be the actual union
    type for user-message content (`ChatCompletionContentPart[]`)
    once the structural distinction is honoured.
  - The empty-text fallback `"[media attached in following user
    message]"` is a model-facing string and should be wrapped in the
    same i18n / system-prompt convention the rest of the converter
    uses for inserted scaffolding.

## Verdict

`needs-discussion`

(Same-shape fix is correct; the *decision* to flip the default vs gate
behind a per-provider flag needs maintainer direction before tests are
rewritten. Author submitted as draft pending exactly that discussion.)
