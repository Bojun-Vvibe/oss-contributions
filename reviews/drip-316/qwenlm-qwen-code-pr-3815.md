# Review: QwenLM/qwen-code #3815 — fix(core): use per-model settings for fast model side queries

- PR: https://github.com/QwenLM/qwen-code/pull/3815
- Head SHA: `ccb52b535c8703269eab51e4673c4438411b20c7`
- Author: B-A-M-N (John London)
- Size: +1128 / -37

## Summary
Fixes #3765 — when side queries (session recap, title generation,
tool-use summary) execute on the fast model, they were reusing the
main model's `ContentGeneratorConfig`, causing `extra_body`,
`samplingParams`, and `reasoning` to leak across models. The fix
resolves the target model's own config via `buildAgentContent
GeneratorConfig()` (the same path subagents use) when
`requestedModel !== mainModel`. **Note**: this PR's diff also carries
the auto-memory-deadline change from sibling PR #3814 — the file
list and ranges are identical, so they appear to share a base branch.

## Specific citations
- `packages/core/src/core/client.ts:46` — adds
  `import { buildAgentContentGeneratorConfig } from
  '../models/content-generator-config.js';` — reusing the subagent
  config builder is the correct factoring.
- `packages/core/src/core/client.ts:1135-1186` (the +52 hunk) —
  introduces the per-model resolution path inside `generateContent`.
  Per the PR body it falls back to the main content generator if the
  model is not in the registry or if creation fails. The fallback
  matters for test environments and unknown models.
- `packages/core/src/core/client.test.ts:3061-3144` — two new tests:
  "should resolve per-model config when the requested model differs
  from the main model" and a negative "NOT called when model matches
  main". The mock `mockResolvedModel` at :3076 sets
  `extra_body: { enable_thinking: false }` and
  `samplingParams: { temperature: 0.1 }`, exactly the leakage case
  from the linked issue.
- `packages/core/src/memory/relevanceSelector.ts:90-92` — drops the
  model-driven selector timeout from 5s to 2s. This is the #3814
  change carried on the branch.
- `packages/core/src/core/client.ts:1117-1137` — the
  `resolveAutoMemoryWithDeadline()` helper races the recall promise
  against a 2.5s deadline, returning `EMPTY_RELEVANT_AUTO_MEMORY_
  RESULT` on timeout. Same #3814 logic.
- `packages/core/src/utils/retry.ts:+166/-9` and
  `packages/core/src/core/geminiChat.ts:+8/-4` — additional retry +
  chat changes are bundled in this PR. The PR body does not
  explicitly call them out — worth a paragraph in the description so
  reviewers know what they're approving.

## Verdict
**needs-discussion**

This PR is presented as a focused per-model-config fix, but the diff
carries (a) the entire #3814 auto-memory-deadline change and (b)
+515/-19 in `retry.test.ts` plus +166/-9 in `retry.ts` that the PR
body does not mention. Either:
- close #3814 in favor of this PR and rewrite this PR's title +
  description to cover all three concerns, or
- rebase this PR onto a branch that excludes the #3814 commits and
  the retry.ts changes, leaving only the per-model fix.

As-is, a reviewer cannot tell from the description what they are
approving on the retry path.
