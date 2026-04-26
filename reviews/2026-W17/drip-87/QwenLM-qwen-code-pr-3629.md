# Review — QwenLM/qwen-code#3629: fix(config): support QWEN_CODE_API_TIMEOUT_MS across OAuth and non-OAuth paths

- **Repo:** QwenLM/qwen-code
- **PR:** [#3629](https://github.com/QwenLM/qwen-code/pull/3629)
- **Author:** B-A-M-N (John London)
- **Head SHA:** `5eb2406c03a851ecac794823b49794721a433b2e`
- **Size:** +377 / −5 across 2 files (`modelConfigResolver.ts` +40/−5, `modelConfigResolver.test.ts` +337/−0)
- **Verdict:** `merge-as-is`

## Summary

Closes a config gap where `QWEN_CODE_API_TIMEOUT_MS` worked only on the API-key auth path and was silently ignored when the user had authenticated via Qwen OAuth. Threads the env-var read through the OAuth branch of `resolveModelConfig`, with `modelProviders` config still taking precedence and validation that rejects non-numeric, negative, and zero values. Test file grows by 337 lines covering 6+ env-var scenarios.

## Technical assessment

This is the kind of PR that's easy to under-rate. The behavior is small (40 lines of impl) but the test coverage is methodical: at `modelConfigResolver.test.ts:188-298` (visible diff window), the new cases exercise (a) `QWEN_CODE_API_TIMEOUT_MS=45000` resolves through OAuth path with `result.sources['timeout'].kind === 'env'` and `envKey === 'QWEN_CODE_API_TIMEOUT_MS'`, (b) `modelProvider.generationConfig.timeout=120000` overrides the env var (precedence test, `kind === 'modelProviders'`), (c) `'not-a-number'` → `timeout` is `undefined`, (d) `'-100'` → `undefined`, (e) `'0'` → `undefined`, (f) `'12345.67'` → `12345` (parseInt-style truncation, not parseFloat — verified by the test's `.toBe(12345)` rather than `.toBe(12345.67)`).

The float-truncation test at `:288-298` is the most informative one. It documents that the parser is `parseInt`-shaped, not `Number()`-shaped, which means `12345.67` becomes `12345` rather than `NaN`. That's a defensible choice (some users will set a value with `.0` from a templating tool) but should ideally be called out in the env-var docs. Not blocking.

The precedence model — `cli > modelProviders > env > settings > default` — is preserved because the OAuth branch now joins the same lookup ladder rather than short-circuiting before the env read. The fact that the resolver carries a `sources` map (each resolved value tagged with `{ kind, envKey }`) and the tests assert against it is exemplary; that's how config debugging stays sane.

## Why merge-as-is

- Pure bugfix: the OAuth path was silently dropping a documented env var, which is a contract violation users would hit unpredictably.
- 337 lines of new test for 40 lines of impl is the right ratio for a config-precedence change.
- The validation rejects non-numeric / non-positive values rather than coercing them to defaults, so misconfigured values become observable rather than silently masked.
- No public API change. No new env var. No new schema field. Only behavior change is "env var that was supposed to work, now works."
- Backed by the same author's prior precision in #3645 (OPENAI_MODEL precedence) and #3648 (ACP repair) — consistent quality from this contributor.

## Optional follow-ups (not blocking)

1. Document the `parseInt`-truncation behavior for `QWEN_CODE_API_TIMEOUT_MS` in the env-var reference (`docs/users/configuration/env.md` or equivalent). One sentence: "Decimal values are truncated to integer milliseconds."
2. Consider a `QWEN_CODE_API_TIMEOUT_S` alias for users who think in seconds; today they have to remember the `_MS` suffix and write `45000` not `45`. Pure ergonomics; orthogonal to this PR.
3. The same precedence-test pattern would be valuable applied to other env vars (`QWEN_CODE_API_BASE`, `QWEN_TLS_INSECURE` from #3635) if not already covered.

## Verdict rationale

`merge-as-is`. Bugfix PR that does exactly what the title says, with test coverage that exceeds the implementation surface and uses the resolver's existing `sources` introspection to verify precedence rather than just observable behavior. No nits worth holding the merge for.
