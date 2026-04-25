# continuedev/continue #12158 — fix(ollama): add Gemma to tool support heuristic for Ollama and LM Studio

- **PR:** https://github.com/continuedev/continue/pull/12158
- **Head SHA:** `5b297fc16697f9585287cb95e82b37c2c7f2ced4`
- **Files changed:** 2 — `core/llm/toolSupport.ts` (+2), `core/llm/toolSupport.test.ts` (+15).

## Summary

Fixes #12131. The Ollama `PROVIDER_TOOL_SUPPORT[ollama]` heuristic in `toolSupport.ts` was missing `"gemma"`, so `modelSupportsNativeTools()` returned `false` for any Gemma 3/4 model running locally. Because LM Studio's heuristic delegates to Ollama's, both providers silently never offered tools to Gemma. The fix is a one-line addition to the substring list, with a code comment linking Google's function-calling docs. Test cases cover `gemma3`, `gemma4`, tagged variants (`gemma3:27b`), and LM Studio's hyphenated `Gemma-3-27B-Instruct-GGUF` form.

## Line-level call-outs

- `core/llm/toolSupport.ts:213-217` — adds `"gemma"` to the Ollama include-list, with the comment `// https://ai.google.dev/gemma/docs/capabilities/function-calling`. Bare substring `"gemma"` is broad enough to accept `gemma2` (Gemma 2 — which does **not** support function calling per Google's own docs; that capability landed in Gemma 3). On Ollama, the secondary `/api/show` template check (added in #11670 per the PR body) will gate per-model — i.e. if the user's Gemma 2 modelfile doesn't include `.Tools`, tools are skipped. So in practice this is fine for Ollama. But on LM Studio (next call-out) there's no such secondary check, and the substring `"gemma"` will say-yes to Gemma 2 regardless.
- `core/llm/toolSupport.ts` (lmstudio branch, around `:280`) — the LM Studio heuristic delegates to Ollama's. There's no `/api/show`-equivalent template check on the LM Studio path. So adding `"gemma"` to Ollama's list flips LM Studio to `true` for *all* Gemma variants, including Gemma 2. If a user loads `gemma-2-9b-it-GGUF` in LM Studio, Continue will now happily send tools to a model that doesn't support tool calling, and they'll get malformed responses. Recommend either: (a) tighten the substring to `"gemma3"`/`"gemma-3"`/`"gemma4"`/`"gemma-4"` (matches the test fixtures already in this PR), or (b) add an explicit `gemma2` exclusion ahead of the substring scan, similar to existing patterns elsewhere in the file.
- `core/llm/toolSupport.test.ts:264-269` — Ollama test. Fixtures exercise `gemma3`, `gemma4`, `gemma3:27b`, `gemma4:27b`. Good. Missing: a negative test that `gemma2:9b` does **not** advertise tool support on a provider where the secondary template check isn't available — the test would fail today as written, which is the point.
- `core/llm/toolSupport.test.ts:299-305` — LM Studio test. Same gap: no `gemma-2-*` negative case. Add `expect(supportsFn("gemma-2-9b-it-GGUF")).toBe(false)` (or, if the consensus is "Gemma 2 doesn't support tools", make it a positive `false` assertion).
- The PR body explicitly notes "the `/api/show` template check (from #11670) will still act as a secondary gate" — but only for Ollama. The LM Studio path is the one with the regression risk and the body doesn't address it. Worth adding a sentence.
- General: code comment links Google's docs page. Good practice; the link is durable (it's part of Google's official Gemma capabilities documentation).

## Verdict

**merge-after-nits**

## Rationale

Right diagnosis, right file, minimal blast radius. The one outstanding concern is the substring `"gemma"` is broader than the actual capability — Gemma 2 doesn't support function calling and the LM Studio path has no secondary check to filter it out. The fix is either to narrow the substring to `"gemma3"`/`"gemma4"` (matches the test fixtures) or add a negative test demonstrating Gemma 2 is correctly handled. Both are five-line changes. Without that, this PR fixes the reported bug for Gemma 3/4 users and silently introduces a smaller bug for Gemma 2 LM Studio users. Tighten and merge.
