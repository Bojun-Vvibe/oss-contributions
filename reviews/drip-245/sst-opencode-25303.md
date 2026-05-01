# PR #25303 — fix: bedrock reasoning issue

- Repo: sst/opencode
- Head: `ed28d72a5d6e1ba37c814720264058b9de5774ee`
- URL: https://github.com/sst/opencode/pull/25303
- Verdict: **merge-after-nits**

## What lands

Closes #24648. When the conversation continues with a *different model* than
the one that produced an earlier reasoning part, opencode was previously
re-emitting the reasoning part with an empty `providerMetadata` (`{}`),
which Bedrock rejects because the reasoning trace is signed and the metadata
is required to validate it. Fix: when crossing model boundaries, drop the
reasoning part entirely and replay its text as plain assistant text instead.

## Specific findings

- `packages/opencode/src/session/message-v2.ts:941-955` — the new branch
  ```
  if (differentModel) {
    if (part.text.trim().length > 0)
      assistantMessage.parts.push({ type: "text", text: part.text })
    continue
  }
  ```
  is the right shape. Previously the code path was
  `...(differentModel ? {} : { providerMetadata: part.metadata })` which kept
  the reasoning part but stripped the metadata — exactly the failure mode
  Bedrock rejects.
- The empty-text guard (`part.text.trim().length > 0`) is correct: some
  providers emit reasoning summaries with empty text (just metadata-bearing
  signatures), and converting those to empty assistant text would produce
  empty content blocks that Bedrock and Anthropic both reject.
- Test coverage at `packages/opencode/test/session/message-v2.test.ts:472-477,
  505` adds a reasoning part `a2` between text `a1` and tool-call `a3`, then
  asserts the merged content is `[{text:"done"},{text:"thinking"},{tool-call}]`
  — the reasoning content is preserved as text in cross-model continuations.

## Nits worth raising before merge

- The new `if (differentModel)` branch isn't covered for the empty-text
  case. The current test only exercises non-empty `"thinking"` text. A
  one-line addition (a reasoning part with `text: ""` or whitespace-only
  asserting it does NOT show up in the converted parts) would lock in the
  guard behavior.
- The branch silently drops `part.metadata` on the floor when crossing
  models. That's the intended behavior per the PR description, but a
  one-line comment at `message-v2.ts:941` explaining *why*
  ("metadata is signed by the producing model and meaningless / harmful when
  replayed under a different one") would help the next reader.

## Verdict rationale: merge-after-nits

The behavioral change is correct and minimal — it fixes a real upstream
rejection. The two nits above are non-blocking but cheap; if the reviewer
wants to ship as-is, the test asymmetry is worth at least flagging in the
PR comments.
