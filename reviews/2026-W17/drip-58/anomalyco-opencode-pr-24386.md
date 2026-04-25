# anomalyco/opencode#24386 — fix(provider): preserve Azure API version

- **Repo:** anomalyco/opencode
- **Author:** pascalandr (Pascal André)
- **Head SHA:** `ee16f5ac` (ee16f5ac3c3334c22cc566a9820a177b75c3b078)
- **Size:** +116 / -1

## Summary
The `@ai-sdk/azure` provider drops the `api-version` query parameter when the
SDK is configured with a custom `baseURL`. This PR injects `api-version` from
`options["apiVersion"]` into the request URL inside the provider's `fetchFn`
wrapper, but only when (a) it isn't already on the URL and (b) the model uses
the Azure npm package.

## Specific references
- `packages/opencode/src/provider/provider.ts:1474-1486` @ `ee16f5ac` — the inline IIFE that computes `requestInput`. Guards on `model.api.npm !== "@ai-sdk/azure"`, non-string `apiVersion`, empty string, and `URL.canParse(input)` — defensive enough.
- `packages/opencode/src/provider/provider.ts:1481-1483` @ `ee16f5ac` — `if (url.searchParams.has("api-version")) return input` — preserves caller-supplied versions. Important for users who pin via baseURL query string.
- `packages/opencode/src/provider/provider.ts:1485` @ `ee16f5ac` — returns `input instanceof URL ? url : url.href`. Preserves the original input shape, so downstream `fetchFn` sees the same type it was given.
- `packages/opencode/test/session/llm.test.ts:678-780` @ `ee16f5ac` — adds Azure-specific stream tests covering the baseURL+api-version case.

## Observations
1. **Header-form `api-version` not handled**: some Azure deployments accept `api-version` via header (`x-ms-version` or similar custom config). The fix is URL-only. Probably fine — the SDK uses query strings — but worth a code comment noting the assumption.
2. **`URL.canParse` is the right primitive**, but on `RequestInfo` of type `Request` (the third allowed input shape), this branch silently no-ops. If `fetchFn` is ever called with a `Request`, the api-version won't be appended. Not in practice today, but a single `else if (input instanceof Request)` branch would future-proof.
3. **Magic string key `"apiVersion"`**: pulled from a stringly-typed `options` bag (`options["apiVersion"]`) rather than the schema-typed Azure model options. If the schema field is camelCase elsewhere too, fine; otherwise a typed accessor would prevent silent misses.
4. The 100-line test addition is good — covers both "user supplied api-version" and "SDK didn't" branches.

## Verdict
**merge-as-is** — narrow fix, well-tested, defensive. Nits are future-proofing, not blockers.
