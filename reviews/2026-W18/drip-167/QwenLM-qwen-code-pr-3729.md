# QwenLM/qwen-code PR #3729 — fix(core): inject reasoning_content on DeepSeek tool-call replays

- Repo: QwenLM/qwen-code
- PR: https://github.com/QwenLM/qwen-code/pull/3729
- Head SHA: `54786ecc02f0df55a32bf561ad7665aad387716a`
- Author: tanzhenxin
- Size: +153 / −35, 3 files
- Closes: #3695, #3619; relates to #3724

## Context

This is a contract-asymmetry bug, the kind that's easy to miss until
production traffic hits the right shape. DeepSeek's thinking-mode endpoint
returns assistant tool-call responses that *sometimes* omit
`reasoning_content` — the model legitimately decided not to reason for
that round and the field is simply absent. On the next turn, when
qwen-code replays the conversation, that prior assistant message goes
back without `reasoning_content`. DeepSeek's thinking mode then rejects
the request: HTTP 400 with `"The reasoning_content in the thinking mode
must be passed back to the API"`. The endpoint emits a shape that, on
round-trip, it then refuses to accept. Classic provider asymmetry.

## What the diff actually does

The fix is provider-scoped (DeepSeek only) and lives in
`packages/core/src/core/openaiContentGenerator/provider/deepseek.ts`.
Two structural changes:

- The existing per-message normalizer in `buildRequest`'s `.map(...)` is
  refactored out into two helpers: `flattenContentParts(message)` (the
  pre-existing array-content → string-content transformation) and a new
  `ensureReasoningContentOnToolCalls(message)`.
- The new helper at the bottom of the file gates on three conditions in
  order: `message.role === 'assistant'`, `Array.isArray(message.tool_calls)
  && message.tool_calls.length > 0`, and finally
  `typeof extended.reasoning_content === 'string' &&
  extended.reasoning_content.length > 0`. If all three pre-conditions for
  "assistant turn with tool_calls and no usable reasoning_content" hold,
  the helper returns a shallow copy with `reasoning_content: ''`.

The type plumbing makes
`ExtendedChatCompletionAssistantMessageParam` (which carries
`reasoning_content?: string | null`) `export`-ed from
`converter.ts:40-43` so the new helper in `deepseek.ts` can cast safely.

## What I like about the fix

1. **The gate is conservative in the right places.** It only fires for
   `role === 'assistant'`, only when `tool_calls` is non-empty, and only
   when `reasoning_content` is missing or empty. Existing valid
   `reasoning_content` is never overwritten — the third predicate is
   the explicit guard at lines 119-124 of the new helper.
2. **The empty-string sentinel is corroborated against a sister
   project.** PR body cites openclaw/openclaw@678ed5d as having shipped
   the same `reasoning_content: ""` workaround against the live DeepSeek
   endpoint. That's the strongest evidence available short of qwen-code
   running its own integration test against `api.deepseek.com`, since
   no source-only test can prove production acceptance.
3. **Three test branches.** `deepseek.test.ts:181-262` covers the
   inject case, the preserve case, and the negative case
   (no `tool_calls` → no injection). Each test uses a tightly scoped
   request shape and asserts the exact field on the right message
   index. Reads cleanly.
4. **Provider isolation.** Other providers go through
   `DefaultOpenAICompatibleProvider` and never see this transformation,
   so the blast radius is exactly the providers that need it. The
   provider-detect logic (per the existing class header — matches
   `api.deepseek.com` base URL or model name containing "deepseek") was
   not changed, which is the right call.

## Things to tighten

1. **The "preserve when present" case should also test `null` and
   whitespace.** The guard is `typeof extended.reasoning_content ===
   'string' && extended.reasoning_content.length > 0`. That correctly
   re-injects `''` when the field is `null` (typeof null === 'object',
   so the guard is false → falls through to overwrite with `''`), but
   also re-injects when the field is `'   '` (length > 0 evaluates
   true, so it preserves whitespace). DeepSeek may or may not accept a
   whitespace-only `reasoning_content`; safer to `extended.reasoning_content.trim().length > 0`,
   and add a test for the `null` and whitespace cases explicitly.

2. **Multi-tool-call rounds aren't tested.** PR body acknowledges
   "multi-tool-call rounds (3+ sequential tools)" as not exercised. The
   helper iterates per-message and the gate doesn't depend on count, so
   it should compose, but a 3-message-chain test would lock that in.

3. **Session resume / compression replay path.** PR body also names
   this as not exercised. If qwen-code's compression rebuilds an
   assistant message from a summary and drops `reasoning_content` in
   the process, the same 400 will fire on the post-compaction first
   turn. Worth grepping for any `.tool_calls` reconstruction outside
   `buildRequest` to confirm `ensureReasoningContentOnToolCalls`
   is actually the only place that needs to enforce this.

4. **`ensureReasoningContentOnToolCalls` is not exported.** Fine for
   the production path, but it makes the unit test rely on
   `provider.buildRequest` rather than testing the helper directly. A
   tiny export (test-only via `__test_only__`) would let future
   regressions be pinned at the helper level rather than at the
   transformer level.

5. **Comment-as-spec.** The block comment at lines 110-116 of the new
   helper is excellent ("DeepSeek's thinking mode requires
   reasoning_content to be replayed on every prior assistant turn that
   carried tool_calls. The model may legitimately return a tool round
   without reasoning text, so the field can be missing when we rebuild
   the request.") — this is exactly the comment that prevents the next
   contributor from "simplifying" the empty-string back out. Leave it
   as-is.

6. **The PR title says "DeepSeek tool-call replays" but the fix
   actually fires whenever the *outgoing* request has a prior assistant
   turn with tool_calls.** Functionally identical for the intended
   use case, but the test names ("injects empty reasoning_content on
   tool-calling assistant turns missing it") are clearer than the PR
   title — worth aligning the title.

## Suggestions

- Use `.trim().length > 0` in the preserve-guard and add tests for the
  `null` / whitespace-only cases.
- Add a multi-tool-call (3 sequential calls) regression test.
- Audit compression / session-resume paths for other places that
  rebuild `tool_calls` messages and confirm they go through this
  helper (or equivalent) before hitting the wire.
- Optionally export `ensureReasoningContentOnToolCalls` for direct
  unit testing.

## Verdict

**merge-after-nits** — the fix is well-scoped, well-commented, and
corroborated against a sibling project's production deployment. The
nits are about hardening the predicate (`null` / whitespace) and
extending the test matrix to multi-call and compression-replay paths.
