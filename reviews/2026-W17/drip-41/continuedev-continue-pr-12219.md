# continuedev/continue PR #12219 — feat(llm): add Doubao (Volcengine Ark) as an LLM provider

- **URL:** https://github.com/continuedev/continue/pull/12219
- **Head SHA:** `27dd211a816ef6191e33491094f938648781cab3`
- **Files touched:** 9 (+222 / -0): provider class, adapter, schema, registrations, docs, tests
- **Verdict:** `merge-after-nits`

## Summary

Adds `provider: doubao` (ByteDance Doubao via Volcengine Ark) as a
first-class LLM provider, mirroring the existing pattern used for
Moonshot/Deepseek/MiniMax. ~30-line OpenAI subclass plus matching
adapter, schema discriminator entry, autodetect templating flag, docs,
and three vitest cases.

## Specific references

- `core/llm/llms/Doubao.ts:18-32` — `Doubao extends OpenAI` with
  `apiBase: "https://ark.cn-beijing.volces.com/api/v3/"`. Intentionally
  sets no `model` default because Ark IDs are date-stamped or endpoint
  IDs (`ep-...`) — author called this out and the rationale is sound.
- `core/llm/autodetect.ts:70` — adds `"doubao"` to
  `PROVIDER_HANDLES_TEMPLATING`. Correct: Ark applies chat templating
  server-side, same as Moonshot/Deepseek above it.
- `packages/openai-adapters/src/apis/Doubao.ts` — thin `OpenAIApi`
  wrapper with the Ark base URL. Matches Moonshot's shape.
- `packages/openai-adapters/src/types.ts` — `DoubaoConfigSchema` slotted
  into the discriminated union; `index.ts` routes
  `constructLlmApi({provider: "doubao"})` to `DoubaoApi`.
- `packages/openai-adapters/src/apis/Doubao.test.ts:+34` — three tests:
  routing, default base URL, OpenAI chat surface preserved.
- `core/llm/llms/OpenAI-compatible.vitest.ts` — Doubao row added to the
  shared subclass matrix. Good defensive coverage.

## Reasoning / nits

- Hard-coding `ark.cn-beijing.volces.com` excludes Ark's other regional
  endpoints (e.g. `ark.ap-southeast-1.volces.com` for SEA users). Worth
  documenting that users with non-Beijing accounts must override
  `apiBase` — or, better, add a one-line note in `doubao.mdx`. Not a
  merge blocker, just a UX miss.
- `maxStopWords = 4` matches Ark's documented limit. Consider linking
  the source in a code comment to make it auditable later.
- No FIM override is correct given Ark has no public FIM endpoint
  today; the comment in the file documents the intent.

Merge after the regional-endpoint doc note.
