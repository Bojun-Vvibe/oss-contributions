# sst/opencode #24635 — feat: Add Dutch (NL) Readme and Crof AI provider support

- PR: https://github.com/sst/opencode/pull/24635
- Head SHA: `7200d8cd945e239c07b264e96a9e6febae68ccbf`
- Author: Yxmura
- Files touched: 23 README locale files, `packages/opencode/test/tool/fixtures/models-api.json`, `packages/ui/src/components/provider-icons/sprite.svg`, `packages/ui/src/components/provider-icons/types.ts`

## Observations

- `README.nl.md` is added as a new locale. The PR also threads `<a href="README.nl.md">Nederlands</a> |` into the language switcher header of every existing `README.*.md` (see e.g. `README.ar.md:32`, `README.bn.md:32`, …) — consistent placement between `العربية` and `Norsk` across all 22 locale files. Mechanical but correctly applied.
- `packages/opencode/test/tool/fixtures/models-api.json` — adds a new top-level `crof` provider entry with `"api": "https://crof.ai/v1"`, `"doc": "https://crof.ai/v1/models"`, and a `models` map (~280 lines). This is a fixture, not runtime config; production provider registration (if any) is not in this diff, so the user-facing impact of "Crof AI provider support" claimed in the PR title is questionable — only the test fixture and an icon are added.
- The `provider-icons/sprite.svg` + `types.ts` change registers a Crof AI logo in the UI sprite sheet. Reasonable scope for a provider-icon contribution.

## Verdict: `request-changes`

**Rationale:** The Dutch README portion is clean and ready to land. But the title says "Crof AI provider support" while the diff only adds a test fixture entry and an icon — there is no actual provider registration / runtime wiring. Either split into two PRs (NL README, Crof icon prep) or add the runtime provider plumbing so the title matches the change. Also worth verifying `crof.ai` is a stable, documented public LLM gateway before adopting its branding into the UI sprite.
