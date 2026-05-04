# Review: sst/opencode #25706 — feat: add FastRouter as an LLM provider

- **PR**: https://github.com/sst/opencode/pull/25706
- **Author**: jatingomnet
- **Base**: `dev`
- **Head SHA**: `fc364109243b803e54af87e87206157b067ce004`
- **Size**: +746 / −7 across 27 files (the bulk being a regenerated `packages/app/.artifacts/unit/junit.xml` and ~20 single-line i18n stubs)

## Scope

Adds FastRouter (`https://go.fastrouter.ai/api/v1`) as a first-class OpenAI-compatible provider, mirroring the existing OpenRouter/LLMGateway integration pattern. Touches the schema, transform layer, login command ordering, web UI provider list, and adds focused unit tests.

## Substantive changes

1. `packages/opencode/src/provider/schema.ts` (line ~23): adds `fastrouter: schema.make("fastrouter")` to `ProviderID`. Correct placement next to the other gateway IDs; no breaking enum change.
2. `packages/opencode/src/provider/provider.ts` (lines ~432–442): registers a `fastrouter` `CustomLoader` with `autoload: false` and the canonical `HTTP-Referer: https://opencode.ai/` + `X-Title: opencode` headers. Pattern matches `openrouter` exactly; safe.
3. `packages/opencode/src/provider/transform.ts`:
   - line ~220: adds `model.providerID !== "fastrouter"` to the interleaved-skip condition. **Correct** — FastRouter, like OpenRouter, sits in front of multiple upstream providers and the gateway handles the interleaved-content shaping itself.
   - line ~265: adds a `fastrouter` entry to the cache-control map (`{ type: "ephemeral" }`), matching OpenRouter semantics.
   - line ~462: `grok-3-mini` reasoning effort gating now also matches `providerID === "fastrouter"`.
   - line ~877 and ~985: `usage.include = true` and `prompt_cache_key = sessionID` are now set when `providerID === "fastrouter"`.
   - line ~1018: `smallOptions` reasoning toggle covers fastrouter.
4. `packages/opencode/src/cli/cmd/providers.ts` (line ~370): inserts `fastrouter: 6` and shifts `vercel: 7` in the login-order map. Order change is purely cosmetic for the login list ordering.
5. `packages/app/src/components/settings-providers.tsx` & `use-providers.ts`: registers fastrouter in the popular-provider set and provider notes map; needs a corresponding `dialog.provider.fastrouter.note` translation key (the i18n stubs in this PR appear to add a single line per locale — worth spot-checking that the stub key name actually matches the lookup).
6. Tests (`packages/opencode/test/provider/{provider,transform}.test.ts`): adds `parseModel handles fastrouter model IDs with slashes` and a complete `ProviderTransform.options - fastrouter` describe block exercising `usage.include`, `prompt_cache_key`, and `smallOptions` for both OpenAI- and Google-routed models. Coverage is appropriate for the size of the surface change.

## Concerns / nits

- **`packages/app/.artifacts/unit/junit.xml` (+618 lines)** is a build artifact that should not be committed. This is the single biggest blocker for "merge as-is" — please add it to `.gitignore` and drop it from this PR.
- The `providers.ts` login-order shuffle (`vercel: 6` → `vercel: 7`) has no functional effect today but if any downstream test asserts on the exact integer index it will need an update. A quick grep for `vercel: 6` would be reassuring.
- The interleaved guard at transform.ts:220 now uses `model.providerID !== "fastrouter"`, but the comparable check elsewhere in the file uses `model.api.npm`. Two parallel ways of identifying the same gateway will drift. Consider centralising "is OpenRouter-style gateway" into a small helper.
- No documentation update under `packages/web/` or any provider docs page — fine if you treat this as a follow-up, but worth calling out.
- The new model IDs (`fastrouter/openai/gpt-5`) imply a slash-separated upstream identifier convention; the existing `parseModel` test confirms that works, but please confirm that downstream cost-tracking tables key off `api.id` rather than `id`, otherwise FastRouter pricing will silently fall through to defaults.

## Verdict

**merge-after-nits** — the integration is mechanically correct and well-tested, but the committed `junit.xml` artifact must be removed before merge, and the duplicate "is gateway" predicate is worth refactoring while we're touching this code path.
