---
pr: https://github.com/sst/opencode/pull/24573
sha: a2e793ea6b8c4d9850af94fa16e44fab7651b901
diff: +4/-0
state: MERGED
---

## Summary

Four-line fix in the provider options builder that forces `toolStreaming = false` when the active model is routed through the `@ai-sdk/google-vertex/anthropic` SDK. Closes upstream #24562 — the Vertex Anthropic backend rejects the streamed-tool-calls protocol that Anthropic-direct accepts, so leaving `toolStreaming` at its default (true) caused tool-call requests to fail.

## Specific observations

- `packages/opencode/src/provider/transform.ts:829-832` — the new branch fires before the existing OpenAI-family `store: false` branch, which is structurally fine because the two flags don't interact and `result` is freshly initialized at `:828`. Branch is keyed on the exact npm package string `@ai-sdk/google-vertex/anthropic` rather than `providerID`, which is correct: Vertex routes other Anthropic models through the same provider id, but only the `/anthropic` sub-package has the streaming-tool-call rejection.
- No regression test in the diff. Given this is a 4-line behavioral switch keyed on a literal string, a single assertion in `transform.test.ts` of the shape `expect(options({ model: { api: { npm: "@ai-sdk/google-vertex/anthropic" }}}).toolStreaming).toBe(false)` would have prevented the obvious silent-revert risk on a future SDK rename — cheap to add.
- The fix is one-directional (force-off) with no escape hatch — if Vertex Anthropic ever fixes its server-side acceptance, this hardcoded `false` will silently degrade users who would benefit from streaming. A `?? false` pattern keyed off an explicit user setting would be more future-proof but is overkill for a closes-the-bug PR.

## Verdict

`merge-as-is` — minimal targeted fix for a concrete provider-side incompatibility with correct branch placement and an exact npm-string match that won't accidentally catch other Vertex models. Already merged.
