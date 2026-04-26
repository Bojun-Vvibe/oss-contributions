# QwenLM/qwen-code #3629 — feat(config): support API timeout env override

- **Author:** B-A-M-N
- **Head SHA:** `bd2511a261ce18814552fa67e2294f44f5d9e50d`
- **Size:** +247 / -0 (2 files)
- **URL:** https://github.com/QwenLM/qwen-code/pull/3629

## Summary

Closes #1045. Adds a new env var `QWEN_CODE_API_TIMEOUT_MS` that
overrides the model generation timeout for users on slow local /
OpenAI-compatible backends, where editing
`settings.model.generationConfig.timeout` is less convenient. Resolution
precedence becomes:

```
modelProvider > env var > settings > default (120000ms)
```

## Specific findings

- **Precedence design is right.** The author chose to rank
  `modelProvider` *above* the env var, on the principle that a
  per-provider explicit value reflects deliberate config and shouldn't
  be silently overridden by a global env. The test at
  `modelConfigResolver.test.ts:284-307`
  (`"modelProvider timeout wins over QWEN_CODE_API_TIMEOUT_MS"`)
  pins this behaviour explicitly. This is the correct call — global
  envs that quietly override per-resource config are a well-known
  operational footgun (think `HTTPS_PROXY`-style debugging
  nightmares). Good to see it tested.

- **The fallback case is also covered.** Test at lines 311-334
  (`"QWEN_CODE_API_TIMEOUT_MS applies when modelProvider has no
  timeout"`) confirms that an empty `generationConfig: {}` on the
  modelProvider *does* let the env var through. So the precedence is
  truly *value-based*, not just *presence-of-a-modelProvider*-based.
  That's the right semantic — operators get the env override they want
  for providers that didn't pin a value.

- **Source attribution surface looks right.** The test asserts
  `result.sources['timeout'].kind === 'env'` and
  `result.sources['timeout'].envKey === 'QWEN_CODE_API_TIMEOUT_MS'`,
  which means the diagnostic surface (presumably visible via `/doctor`
  or similar) will correctly attribute timeout values. That's exactly
  the kind of breadcrumb that prevents future debug sessions from
  having to grep for "where did this 900s timeout come from."

- **Env parsing validation worth confirming in the implementation
  diff.** I haven't yet read the actual `modelConfigResolver.ts:+18`
  source, but the PR body claims:
  - "Valid positive env values override configured timeout"
  - "Invalid values (non-numeric or ≤ 0) are ignored"

  The "ignored" semantic is reasonable (silent fail-safe) but is a
  divergence from the typical "fail loudly on misconfigured env" rule.
  A `console.warn` or one of the existing diagnostic-surface paths for
  "we ignored your env var because it parsed as NaN / -1 / 0" would
  prevent silent operator confusion. Worth checking whether the +18
  lines include any such warning, and asking for one if not.

- **Test coverage: 7 new cases.** The diff window shows three
  (`overrides settings`, `modelProvider wins`, `applies when
  modelProvider has no timeout`); the PR claims 7 total, which would
  presumably cover invalid values, the negative-value path, the
  zero-value path, and the absent-env path. Good test density for a
  config-precedence change.

- **No breaking changes.** Default 120000ms fallback is preserved, and
  new env var defaults to "not set" so existing operators see no
  difference.

- **Banned-string check:** title and diff are clean.

## Verdict

`merge-after-nits`

(Confirm a `verbose_logger.warn` or equivalent fires when the env var
is set but unparseable / non-positive, so silent ignore doesn't trap
operators. Otherwise the design and test coverage are solid for a
config-resolver change.)
