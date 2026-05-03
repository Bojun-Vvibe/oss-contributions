# Review — QwenLM/qwen-code#3815 — fix(core): use per-model settings for fast model side queries

- PR: https://github.com/QwenLM/qwen-code/pull/3815
- Head: `ccb52b535c8703269eab51e4673c4438411b20c7`
- Verdict: **merge-after-nits**

## What the change does

Wires `Config.getModelsConfig().getResolvedModel(model)` into the
`generateContent`-with-fast-model code path in `packages/core/src/core/client.ts`,
so that when a request targets a model different from the main model
(e.g. a fast/side model for auto-memory or summarization) the per-model
`extra_body`, `samplingParams`, and auth-type settings are honored
instead of inheriting the main model's. Test
`packages/core/src/core/client.test.ts::"should resolve per-model config
when the requested model differs from the main model"` covers a fast
model with `enable_thinking: false` and `temperature: 0.1`.

Two adjacent test additions also exercise the existing auto-memory-recall
deadline behavior:

- `"should not block the main request when auto-memory recall is slow"`
  uses `vi.useFakeTimers` + `advanceTimersByTimeAsync(3_000)` and asserts
  the main request fired with no slow-memory string in the prompt.
- `"should include auto-memory prompt when recall completes within deadline"`
  asserts the fast-recall result is folded in.

## Strengths

- Test mocks add `getModelsConfig: vi.fn().mockReturnValue({
  getResolvedModel: vi.fn().mockReturnValue(undefined) })` to the existing
  `mockConfig`, preserving back-compat for tests that don't care about
  per-model resolution.
- The slow-recall test uses fake timers correctly: `advanceTimersByTimeAsync`
  before `await streamPromise`, so the 2.5 s deadline fires deterministically.
- `expect.not.arrayContaining([...])` neatly asserts "slow memory was
  excluded", which is the right negative shape for a deadline test.

## Nits

1. The mocked resolved model uses `authType: 'openai' as const` —
   confirm there's a follow-up assertion (or another test) that the
   resolved auth type actually changes the downstream call shape; the
   excerpt in the diff stops before that assertion.
2. `getResolvedModel` is now called on every `generateContent` request
   to a non-main model. If `ModelsConfig` does any I/O (file read, JSON
   parse) per call, memoize.
3. Slow-recall test assertion `expect.not.arrayContaining([
   expect.stringContaining('Slow memory result') ])` only fires if the
   recall string is passed as one element of the array. If it's
   concatenated into a larger string, the matcher silently passes — a
   `expect.not.stringContaining(...)` on each element would be stricter.

Mergeable after spot-checking the ResolvedModel I/O cost.
