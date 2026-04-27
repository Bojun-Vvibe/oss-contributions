# sst/opencode#24612 — feat: dynamically load model list from LM Studio

- PR: https://github.com/sst/opencode/pull/24612
- Head SHA: `36157b68`
- Diff: +136 / -1, two files (new `provider/model-cache.ts`, edits to `provider/models.ts`)

## What it does
Adds `ModelCache.fetch(providerID, { baseURL, apiKey })` (`model-cache.ts:8-30`) which, for `providerID === "lmstudio"` or `"apertis"`, calls the OpenAI-compatible `GET {baseURL}/models` endpoint and converts each entry into the internal model schema. Result is cached per-providerID in a module-level `Record`. `models.ts` is wired to call this when LM Studio is detected so the user sees the actual local models instead of the static `models.dev` triple.

## Strengths
- Sensible default `baseURL` of `http://127.0.0.1:1234/v1` matches LM Studio's documented default; explicit `AbortSignal.timeout(10_000)` (`model-cache.ts:48`) prevents UI hangs when LM Studio isn't running.
- Cleanly factored: `fetchOpenAICompatibleModels` is reused for both `lmstudio` and `apertis` (`model-cache.ts:84`), so future OpenAI-compatible self-hosted providers can plug in with a one-line case in the dispatch.
- Failure mode is "return `{}`" (`model-cache.ts:55`, `:75`) — opencode falls back to the static models.dev list, which is exactly what users expect.

## Concerns
- **Hardcoded limits at `model-cache.ts:62-66`**: `context: 128000`, `output: 4096`, `attachment: true`, `reasoning: false`, `tool_call: true`, modalities `["text","image"] -> ["text"]`. These are wrong for many LM Studio models (a 7B Llama variant doesn't have 128k context, doesn't accept images, may not support tool calls). Showing the user a model card that lies about capabilities will lead to runtime errors that look like opencode bugs. At minimum, set conservative defaults (`context: 8192`, `attachment: false`, `tool_call: false`) and let users override via config. Even better, query LM Studio's `/v1/models/{id}` for capability metadata when available.
- **Cache is process-lifetime, no invalidation**: `cache[providerID]` (`model-cache.ts:9`) is populated on first fetch and never cleared. If the user loads a new model in LM Studio mid-session, opencode keeps showing the stale list. `refresh()` exists (`:23`) but nothing in this PR calls it. Suggest at minimum a TTL (e.g. 60s) or invalidation on user-triggered "refresh models" action.
- **Apertis hardcoded URL** at `model-cache.ts:33` (`https://api.apertis.ai/v1`) ships a third-party endpoint as part of the PR with no mention in the description (which only talks about LM Studio). This deserves its own PR or at least a sentence in the description explaining why Apertis is in scope.
- **`Bun.fetch` in shared code**: hard-bakes the Bun runtime. Most of opencode is bun-only already, but if any non-bun consumer (electron, worker) imports `provider/model-cache.ts`, it breaks. Use the existing HTTP helper if there is one.
- **Error log includes `apiKey` indirectly**: `log.error("...", { url, error })` (`model-cache.ts:75`) — the `error` object from `Bun.fetch` typically includes the request URL but not the bearer token, so probably safe, but worth confirming.

## Verdict
**request-changes** — the feature is wanted and the structure is right, but the hardcoded capability fields and the unannounced Apertis addition need to be addressed before merge. The cache invalidation story is the biggest UX issue.
