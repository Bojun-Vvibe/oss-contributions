# Review: anomalyco/opencode #25971 — fix(opencode): recover malformed GLM/NVIDIA tool calls

- Carrier: `anomalyco/opencode`
- PR: #25971
- Head SHA: `80416b1c`
- Drip: 387

## Verdict
`merge-after-nits` (man)

## Summary
A 398-line vendored patch against `@ai-sdk/openai-compatible@2.0.41` that
hardens both the streaming and non-streaming OpenAI-compatible chat code
paths against three real-world malformations seen from GLM-4.5/4.6 and
NVIDIA's NIM-hosted models:
1. Non-streaming `choices[0].message.function_call` (legacy OpenAI shape)
   without a parallel `tool_calls` array — now synthesized into a
   `content.push({type:"tool-call", toolCallId: generateId(), ...})`.
2. Streaming `delta.function_call` (also legacy) — now lifted into a
   synthetic single-element `tool_calls` delta with `index: 0`.
3. Streaming `delta.tool_calls[i]` arriving with `function.name == null`
   on early frames (NVIDIA's `kimi-k2.5` does this) — now buffered into a
   `pendingToolCallIds[index]` map so the eventual frame carrying the name
   resolves the same generated `toolCallId` instead of throwing
   `InvalidResponseDataError`.

Plus a fallback-on-empty path: if `finishReason === "tool-calls"` ends the
stream but `sawToolCallStart === false`, the patch re-issues the same
request without `stream:true` and synthesizes the tool-call frames from the
non-streaming response. If the second call also yields nothing nameable, an
explicit `error` chunk with `raw: "tool_calls_missing_name"` is emitted
instead of silently dropping the turn.

## Diff anchors
- `patches/@ai-sdk%2Fopenai-compatible@2.0.41.patch:611-619` (dist/index.js)
  — non-streaming `function_call → tool-call` synthesis at the
  `OpenAICompatibleChatLanguageModel.doGenerate` content-assembly site.
- `:644-674` — `doStream` now hoists `requestUrl` / `requestHeaders` /
  `fallbackBody` / `failedResponseHandler` / `fetch` to outer scope so the
  flush-time fallback POST can replay the exact same request shape.
- `:755-786` — `toolCallDeltas = delta.tool_calls ?? (delta.function_call
  ? [{index:0, id:undefined, function: delta.function_call}] : undefined)`
  unifies the two wire formats at a single switch point.
- `:786-805` — replaces the previous twin `throw InvalidResponseDataError`
  on missing `id` and missing `function.name` with a `pendingToolCallIds`
  buffer + `generateId()` synthesis. The `delete pendingToolCallIds[index]`
  on resolve is correctly placed inside the name-arrival branch only.
- `:880-955` — the `flush(controller)` rewrite from sync to `async` is the
  load-bearing change: on `finishReason.unified === "tool-calls" &&
  !sawToolCallStart`, it re-POSTs `fallbackBody` (without `stream:true`),
  reads `choices[0].message.tool_calls ?? (function_call ? [...] : [])`,
  emits the synthesized `tool-input-start`/`tool-input-delta`/
  `tool-input-end`/`tool-call` quartet per fallback tool, and falls through
  to a final `error` chunk if even the non-streaming response is empty.
- A symmetric edit at `dist/index.mjs:598-` mirrors the CJS path so the
  ESM build benefits identically.

## Concerns / nits
1. **No upstream coordination noted in PR body.** Vendoring a 398-line
   patch against a third-party npm package is acceptable when the upstream
   maintainer is unresponsive, but the PR doesn't link an upstream issue
   or PR. A one-line `// upstream: vercel/ai#<n>` comment in the patch
   header would make later reviewers' jobs much easier when the package
   bumps to 2.0.42.
2. **`pendingToolCallIds` is a plain object keyed by `index`.** If the
   provider sends `index: -1` (legal for OpenAI under the chat-completions
   spec when the field is absent and falls through to `toolCalls.length`),
   the bookkeeping is fine, but a `Map<number, string>` would document
   intent better and survive a future `Object.create(null)` audit.
3. **Fallback retry doubles the request budget.** If the first stream
   exhausts a per-key rate limit and the fallback fires, the user sees
   a single visible request but two backend hits. Worth a metric or at
   minimum a `debug!` log line so this is observable in retro on incidents.
4. **`InvalidResponseDataError` is now never thrown for these cases.**
   Downstream consumers that pattern-matched on that error class for
   "this provider is broken, try another" UX silently lose that signal.
   The new `error` chunk's `raw: "tool_calls_missing_name"` is a fine
   replacement but should be documented in CHANGELOG.
5. **No new bun:test in this diff.** A patch this large against a real
   bug class deserves at least three fixtures (legacy `function_call`
   non-streaming, legacy `function_call` streaming, deferred-name
   streaming) — the author may be relying on the upstream package's test
   suite but the patch isn't reapplied during CI without its own assertions.
6. The `delete pendingToolCallIds[index]` at `:797` is correct for the
   "name arrived after id" path, but if the same `index` is re-emitted
   *after* `tool-input-start` fires (which providers shouldn't do but
   might), the second visit would see no `pendingToolCallIds[index]` and
   would call `generateId()` again. Defensive but worth documenting.

## Risk
Medium. The patch is large, the bug class is real and high-frequency on
two named providers (GLM-4.5/4.6, NVIDIA NIM), and the fallback path adds
network-request amplification. The changes preserve the happy path
byte-for-byte (same `controller.enqueue` shapes, same `finishReason`
resolution) so well-behaved providers are unaffected. Upstreaming and
test coverage are the two outstanding items before merge.
