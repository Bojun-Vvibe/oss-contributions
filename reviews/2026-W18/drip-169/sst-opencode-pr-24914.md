# sst/opencode#24914 — `feat(provider): add Chat Completions API mode for gateway/proxy providers`

- PR: https://github.com/sst/opencode/pull/24914
- Head SHA: `9a4aeb6f164cf49126ad634575ca3a5671c8cb13`
- Author: darthbeeius

## Summary of change

Introduces a config-driven API-mode selector. When a provider config
sets `"api": "chat"` (or `"responses"`), opencode now (a) treats it as
a *mode*, not as a base URL, (b) calls `sdk.chat(modelID)` /
`sdk.responses(modelID)` instead of the default `sdk.languageModel()`,
and (c) rewrites the outbound HTTP body so GPT-5.x via gateways/Bedrock
stops choking on `max_tokens` and `$ref` in tool params. Three new
tests in `packages/opencode/test/provider/provider.test.ts` exercise
the config-side parsing.

## Specific observations

- `packages/opencode/src/provider/provider.ts:141-158` — new
  `resolveLanguageModel(sdk, modelID, apiMode)` is fine but silently
  falls back to `sdk.languageModel()` when the requested method
  doesn't exist (`typeof sdk.chat !== "function"`). For an SDK that
  just doesn't expose `.chat`, this is a *quiet downgrade* to
  whatever the default is — which may not be Chat Completions at all.
  At least a `console.warn` (or an `Effect.log`) so the user notices
  their explicit `"api": "chat"` got ignored.
- `provider.ts:1168-1176` — `API_MODES = ["chat", "responses"]` is
  declared inline inside the loop body. Hoist it to module scope; it
  also needs to be the single source of truth that the
  URL-fallthrough on line 1204 reads from (currently the same array
  literal is inlined twice in spirit).
- `provider.ts:1201-1208` — the `iife` wrapper around the URL
  fallthrough is correct but reads awkwardly. A named helper
  (`resolveModelApiUrl(model, provider, existingModel, modelsDev)`)
  with the `API_MODES.includes()` guard inside would make the
  precedence chain (`model.provider?.api > provider?.api(if URL) >
  existing > modelsDev > ""`) much easier to scan.
- `provider.ts:1537-1561` — the body rewrite parses `opts.body` with
  `JSON.parse` unconditionally. If a caller ever passes a non-JSON
  body (FormData, raw bytes, an already-parsed object) under
  `apiMode === "chat"`, this throws inside the interceptor. Either
  guard with `typeof opts.body === "string"` or wrap in a try/catch
  that logs and falls through. Same applies to the `$ref` resolver
  walking arbitrary user JSON without a depth bound — `resolveRefs`
  on a self-referential schema will recurse forever.
- `resolveRefs` (lines 161-178): the `JSON.parse(JSON.stringify(...))`
  deep-clone of every resolved `$def` per occurrence is wasteful for
  any tool with a recursively-shared subschema. Acceptable for now
  given tool counts are small, but worth a TODO for a memoised
  resolver if this ever runs hot.
- The `delete params["defs"]` on line 1559 is suspicious — the JSON
  Schema key is `$defs`, never `defs`. Either it's defensive against
  a non-standard producer, or it's a typo and the real cleanup is
  only the `delete params["$defs"]` one line above. Worth a comment
  if intentional, removal if not.
- Test at `provider.test.ts:2714-2758` correctly asserts the
  invariant the bug originally violated (`model.api.url` is *not*
  the literal string `"chat"`). Good regression coverage. The
  third test (`"api URL string ... still flows ... normally"`)
  closes the back-compat side. Missing: an end-to-end body-rewrite
  test that round-trips through the interceptor.

## Verdict

`merge-after-nits` — the design is sound and the parsing/URL
plumbing is clearly the right shape. The `JSON.parse(opts.body)` on
the hot path needs a guard, the `console.warn` on silent SDK
downgrade should land before merge, and the inline `API_MODES`
literal is an obvious cleanup.
