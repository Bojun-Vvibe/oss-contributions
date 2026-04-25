# cline/cline #10403 — feat: add Abliteration.ai provider

- **Repo**: cline/cline
- **PR**: [#10403](https://github.com/cline/cline/pull/10403)
- **Head SHA**: `975c5cdfabdabd29fbdc62a6e1be8f088b28481c`
- **Author**: abliteration-ai
- **State**: OPEN (+344 / -6)
- **Verdict**: `request-changes`

## Context

Adds Abliteration.ai (`https://api.abliteration.ai/v1`) as a
first-class provider on the order of OpenAI / Anthropic /
Vertex / etc. Wires the OpenAI-compatible API in via a new
`AbliterationHandler`, registers a single
`abliterated-model` with documented metadata (150K context,
streaming, tool calling, vision), and adds settings UI, API
key validation, configured-provider detection, secret
storage, provider defaults, proto conversions, and unit
coverage.

## Design

The change spans 16 files; the substantive ones are:

1. **Handler implementation**
   (`src/core/api/providers/abliteration.ts:1-105`). New
   class `AbliterationHandler` implements `ApiHandler`, uses
   `createOpenAIClient` against the documented base URL, and
   delegates streaming chunk processing to the existing
   `ToolCallProcessor`. The streaming loop at lines 49-71
   handles `delta.content`, `delta.tool_calls`, and the
   trailing `chunk.usage` block. `getModel()` falls back to
   `abliterationDefaultModelId` when `apiModelId` is not in
   `abliterationModels`. `supportsImages()` reads from the
   model info (correct), `supportsTools()` returns hardcoded
   `true` (see Risks below).

2. **Model registry** (`src/shared/api.ts:1465-1479`):

   ```ts
   export const abliterationModels = {
     "abliterated-model": {
       maxTokens: -1,
       contextWindow: 150_000,
       supportsImages: true,
       supportsPromptCache: false,
       supportsTools: true,
       supportsStreaming: true,
       inputPrice: 3.0,
       outputPrice: 3.0,
       temperature: 0,
       description: "...",
     },
   } as const satisfies Record<string, OpenAiCompatibleModelInfo>
   ```

   `maxTokens: -1` is the red flag here — see Risks. The
   pricing fields (`$3 / $3 per ??`) are also under-specified
   in the metadata: the field-level convention elsewhere in
   `api.ts` is "USD per million tokens", but a downstream
   reader has no way to know that without grepping for
   `inputPrice` usages.

3. **Provider plumbing**: `src/core/api/index.ts:153-159`
   adds the `case "abliteration":` branch wiring
   `abliterationApiKey` and `apiModelId` from the resolved
   options. The provider key list at
   `src/shared/api.ts:11` adds `"abliteration"` between
   `"openai"` and `"ollama"`.

4. **Proto conversions**
   (`src/shared/proto-conversions/models/api-configuration-conversion.ts:257-258`)
   maps `"abliteration"` ↔ `ProtoApiProvider.ABLITERATION`.
   The matching `proto/cline/models.proto` and
   `proto/cline/state.proto` edits register the enum value.
   This is the correct shape for cline's typed wire — needs a
   matching codegen run before merge (the PR does include the
   generated outputs).

5. **UI / settings**:
   `webview-ui/src/components/settings/providers/AbliterationProvider.tsx`
   (new, 50ish lines) for the API-key input;
   `ApiOptions.tsx` for the dropdown entry; `providerUtils.ts`
   for default-model lookup; `getConfiguredProviders.ts` for
   the "is this provider configured?" check;
   `provider-keys.ts` and `state-keys.ts` for secret /
   state storage. All follow the existing patterns —
   nothing surprising.

6. **Tests**
   (`src/core/api/providers/__tests__/abliteration.test.ts:1-110`)
   cover default-model fallback, OpenAI client construction
   with the right `baseURL`, the streaming-content yield
   shape, and the cost calculation through
   `calculateApiCostOpenAI`.

## Risks

- **`maxTokens: -1`** at `api.ts:1466`. Searching the rest
  of `api.ts` for the convention: most providers use a
  positive integer (e.g., OpenAI's `gpt-4o`: `maxTokens:
  16384`). `-1` is being used here as a sentinel meaning
  "unbounded" / "not specified". If the rest of the cline
  pipeline interprets `maxTokens` as an upper-bound on the
  model's response length and respects the value (e.g.,
  passing `max_tokens: -1` into the OpenAI request), most
  OpenAI-compatible servers will reject `-1` with a 400. The
  PR doesn't pass `max_tokens` to `chat.completions.create`
  (the call at handler.ts:48 omits it), which avoids the
  immediate failure, but downstream cost / context-budget
  calculations that read `model.info.maxTokens` will be
  wrong (negative budget). Either set this to a real
  positive integer (the model's documented max output
  tokens) or document `-1` in the type as a sentinel and
  audit every consumer. **Blocking.**

- **`supportsTools(): boolean { return true }`** hardcoded
  at handler.ts:96. The model registry already has
  `supportsTools: true` at api.ts:1471 — the handler should
  read from `this.getModel().info.supportsTools === true`,
  matching the `supportsImages()` shape on the line above.
  Otherwise, when a future model is added with
  `supportsTools: false`, the handler will lie and the
  agent will issue tool calls the backend rejects.

- **First-party-provider provenance.** This PR is opened by
  the user `abliteration-ai`, against `cline/cline`,
  promoting their own service to first-class status alongside
  OpenAI / Anthropic / Vertex. The cline project should have
  a documented bar for first-class providers (terms of
  service, uptime SLA, abuse policy, who-to-call when the
  API breaks the contract). If the bar is "anyone with an
  OpenAI-compatible endpoint can be first-class," that's
  fine — but worth a maintainer policy doc rather than
  case-by-case decisions.

- **No retry-policy specification.** The handler uses
  `@withRetry()` without arguments; the default policy
  should be confirmed appropriate for a provider whose
  rate-limit / 429 behaviour is unknown to cline. If the
  retry helper has a documented default (e.g., exponential
  backoff with 5 attempts), fine; if it's unbounded, this
  is a footgun.

- **API-key validation.** The PR mentions "API key
  validation" in the description but the diff doesn't show
  a key-format check (e.g., regex on length / prefix). Most
  providers' API keys have a stable prefix (`sk-...`,
  `xai-...`); without a format check, the user can paste a
  random string and get a confusing failure on first request
  rather than at config-save time. Optional, but standard.

## Suggestions

- **Replace `maxTokens: -1` with the documented max-output
  token count for the model.** If unbounded is the actual
  behaviour, document the `-1` sentinel in the
  `OpenAiCompatibleModelInfo` type and audit every consumer.
- **Read `supportsTools` from the model info**, mirroring
  `supportsImages`.
- Add a maintainer policy doc for first-class provider
  acceptance (separate PR).
- Confirm `@withRetry()` default policy is appropriate;
  consider per-provider override.
- Consider adding an API-key format check at config-save
  time.

## Verdict reasoning

`request-changes`. The handler shape and proto wiring are
clean, but `maxTokens: -1` is a real correctness/contract
issue that will surface as bad cost calculations downstream,
and the hardcoded `supportsTools` is a future-bug magnet.
Both are 1-line fixes. The first-party-provider provenance
question is a maintainer policy call, not a code-review
blocker.

## What I learned

The "I'll add my own service as a first-class provider" PR
shape is a recurring one in coding-agent ecosystems, and the
review tension is the same every time: mechanically the diff
follows the existing pattern (great, it shows the contributor
read the code), but the substance is "should this service be
in the dropdown next to OpenAI?" — which is a maintainer
policy question dressed up as a code review. The healthiest
shape is for the project to publish the bar (e.g., "must
have a public terms of service, must be reachable for
abuse complaints, must keep the API stable") so the PR
process can stay focused on the diff. Without that, every
new-provider PR re-litigates the policy from scratch and
the maintainer is the bad guy. The `maxTokens: -1` issue is
a separate, mechanical bug that shouldn't get lost in the
larger conversation.
