# sst/opencode #25721 — fix: ensure anthropic sdk properly resolves when using azure

- PR: https://github.com/sst/opencode/pull/25721
- Head SHA: `0f06e74b8c667eecc5dce232fb692e70ac34a629`
- State: merged
- Diff size: +10 / -12 across 1 file

## Summary

Refactors the SDK-method-selection logic in two custom provider
loaders (anthropic-on-azure and another similar provider) into a
single `selectAzureLanguageModel` helper. The new helper adds
`sdk.messages(modelID)` and `sdk.chat(modelID)` as additional
fallback paths after `sdk.responses(modelID)`, which is what makes
the anthropic SDK (which exposes `.messages` rather than `.responses`)
work behind the same loader.

## Citations

- `packages/opencode/src/provider/provider.ts:141-147` — new helper.
  Selection order:
  1. If caller explicitly asked for chat (`useChat && sdk.chat`) →
     `sdk.chat(modelID)`.
  2. Else `sdk.responses(modelID)` if present.
  3. Else `sdk.messages(modelID)` if present (anthropic SDK shape).
  4. Else `sdk.chat(modelID)` (catch-all).
  5. Else `sdk.languageModel(modelID)` (legacy).
  This is sane, but the precedence "responses before messages" means
  if a hypothetical SDK exposes both (e.g. an SDK that wraps both
  `/responses` and `/messages` for parity), the responses path wins
  silently. Probably what you want for openai-compatible flows, but
  worth a brief comment explaining the priority.
- `provider/provider.ts:233, 253` — both call sites collapsed from
  a 5-line conditional to a single `return selectAzureLanguageModel(...)`.
  Net deletion of duplicated logic, which is the right cleanup.
  Note that the *old* code path used `useLanguageModel(sdk)` as an
  early-out which was equivalent to "if sdk has neither `.responses`
  nor `.chat`, use `.languageModel`". The new helper checks
  `.messages` *before* falling through to `.languageModel`, so
  SDKs that expose only `.messages` + `.languageModel` (anthropic)
  will now hit the `.messages` branch instead of `.languageModel`.
  This is the actual bugfix.
- The helper is typed as `(sdk: any, modelID: string, useChat:
  boolean)` — losing all SDK shape typing. The original code was
  also `any`-typed so this is not a regression, just a missed
  opportunity to introduce a `LanguageModelSDK` interface union.

## Risk

Low. Behavior change is strictly additive: paths that previously
worked still work; the `.messages` SDK now works where it didn't
before. Already merged, so risk assessment is academic.

## Verdict

`merge-as-is` — small, correct, removes duplication, and fixes a
real anthropic-SDK shape mismatch. Already merged. The follow-up
opportunity (typed SDK union) is not blocking.
