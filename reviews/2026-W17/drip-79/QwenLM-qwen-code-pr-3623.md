# QwenLM/qwen-code PR #3623 — fix(cli): recognize OpenAI-compatible providers in `qwen auth status`

- **PR:** https://github.com/QwenLM/qwen-code/pull/3623
- **Author:** doudouOUC (jinye)
- **Head SHA:** `bf49da22fc49585f867ccdc912c25b4886390904`
- **Files:** 2 (+298 / -?)
- **Verdict:** `merge-after-nits`

## What it does

The `qwen auth status` command treated *any* `selectedType === USE_OPENAI` as the Alibaba Cloud Coding Plan path. Users running against generic OpenAI-compatible endpoints (Xunfei, DeepSeek, Ollama, etc.) saw a misleading `⚠️  Authentication Method: Alibaba Cloud Coding Plan (Incomplete)` even though their setup was perfectly valid. The patch splits the `USE_OPENAI` branch into two paths gated on whether `codingPlan.region` or `CODING_PLAN_ENV_KEY` is present.

## Specific reads

- `packages/cli/src/commands/auth/handler.ts:33-36` — the new detector:
  ```ts
  const hasCodingPlanKey =
      !!process.env[CODING_PLAN_ENV_KEY] ||
      !!mergedSettings.env?.[CODING_PLAN_ENV_KEY];
  ```
  Then `if (codingPlanRegion || hasCodingPlanKey)` selects the Coding Plan branch; everything else falls into the new generic branch. Reasonable disambiguator — `CODING_PLAN_ENV_KEY` is unique to the Alibaba flow and `codingPlan.region` is only set by `qwen auth coding-plan`.
- `packages/cli/src/commands/auth/handler.ts:117-133` — generic branch hunts the API key across four sources, in priority order:
  ```ts
  const hasApiKey =
      !!(modelConfig?.envKey &&
         (process.env[modelConfig.envKey] || mergedSettings.env?.[modelConfig.envKey])) ||
      !!process.env['OPENAI_API_KEY'] ||
      !!mergedSettings.env?.['OPENAI_API_KEY'] ||
      !!mergedSettings.security?.auth?.apiKey;
  ```
  This mirrors how the OpenAI provider actually resolves keys at request time, so status accurately reflects the runtime check.
- `packages/cli/src/commands/auth/handler.ts:146-150` — `baseUrl` is surfaced for the generic provider; this is genuinely useful UX for users who can't remember which Ollama instance they pointed at.
- `packages/cli/src/commands/auth/status.test.ts:144-200` — new test covers the Coding-Plan-via-env-key case (no region) and asserts the output does *not* contain `OpenAI-compatible Provider`. Good negative-assertion discipline.

## Risk surface

Low. The change is confined to one display function (`showAuthStatus`) and its test file. No production auth code path changes — only the human-readable status summary. The widened `MergedSettingsWithCodingPlan.security.auth` type adds optional `apiKey` / `baseUrl` strings; backward compatible.

One edge case: if a user *both* has `CODING_PLAN_ENV_KEY` set in their shell *and* configured a generic OpenAI provider in settings, this code reports Coding Plan. That matches the actual auth resolver behavior (Coding Plan env wins) so the status stays truthful.

## Nits (not blocking)

1. The new path duplicates a lot of `writeStdoutLine(t('  Current Model: ...'))` style strings between the Coding Plan and generic branches. A small `printCommonStatus(modelName)` helper would shrink the function and make future status fields land in both branches automatically.
2. The generic branch only asserts in tests via the existing Coding-Plan negative test — there is no positive test covering "generic OpenAI provider with `OPENAI_API_KEY` set → prints OpenAI-compatible Provider". Worth adding two cases (env key + settings key) to lock in the four-source fallback chain.
3. The `(Incomplete)` label is reused for both branches but the suggested remediation differs (`qwen auth coding-plan` vs `qwen auth`). Consider tagging which provider type is incomplete in the warning header itself, e.g. `⚠️  Authentication Method: OpenAI-compatible Provider (no API key found)` — easier for users to grep in support tickets.

Verdict: merge after the helper extraction and one positive test for the generic branch — the core logic is correct and the bug fix is real, but the surface area added warrants tighter test coverage before landing.
