# QwenLM/qwen-code #3788 — fix(core): inject thinking blocks for DeepSeek anthropic-compatible provider

- **Head SHA:** `ddaedeba1b945beda37c500bbd012826f2fc7675`
- **Files:** `packages/core/src/core/anthropicContentGenerator/anthropicContentGenerator.ts` (+18), `anthropicContentGenerator.test.ts` (+133), `converter.ts` (+45), `converter.test.ts` (+179)
- **Verdict:** `merge-after-nits`

## Rationale

Targeted compatibility shim for DeepSeek's anthropic-compatible API (issue #3786 referenced in the test comment). The DeepSeek endpoint rejects subsequent requests when any prior assistant turn omits a `thinking` block while thinking mode is on, so the converter now prepends `{ type: 'thinking', thinking: '', signature: '' }` to assistant turns when the provider is detected. Detection is dual: `baseUrl` containing `api.deepseek.com` OR model name matching `deepseek-*`, validated by the two test cases at `anthropicContentGenerator.test.ts:497` and `:541`. The negative test at `:583` confirms non-DeepSeek providers (claude on `api.anthropic.com`) are untouched — the right invariant.

Nits: (1) Model-name detection via substring `deepseek` is fragile — a user model alias like `my-deepseek-proxy` triggers it. Worth a comment in `converter.ts` documenting the heuristic and recommending users set `baseUrl` for reliability. (2) The empty `signature: ''` is necessary for Anthropic's schema but worth noting in a comment that real signatures are not required by DeepSeek's variant — otherwise a future reader will think this is a bug. (3) Confirm the injected empty thinking block doesn't bleed into the response stream (i.e. the user never sees a phantom empty thinking block in transcripts).

312 lines of new test coverage for ~63 lines of production code is a strong ratio. Merge once the heuristic is documented inline.

