---
pr_number: 26419
repo: BerriAI/litellm
head_sha: 3bc6dd69987800e0eb159dd9dca76dedcb3b900a
verdict: merge-as-is
date: 2026-04-24
---

# BerriAI/litellm#26419 — add missing `zai` (Z.AI / Zhipu AI) provider to Add-Model dropdown

**What changed.** Three additions in `ui/litellm-dashboard/src/components/provider_info_helpers.tsx`:

1. `Providers.ZAI = "Z.AI (Zhipu AI)"` enum entry (line 106).
2. `provider_map.ZAI = "zai"` (line 214) — the backend's canonical lowercase value matches what `/public/providers` already returns.
3. `getPlaceholder(Providers.ZAI)` returns `"zai/glm-4.5"` (line 371), parallel to the watsonx/cursor entries above it.

Tests in `provider_info_helpers.test.tsx` add two cases (lines 71–80, 169–171): one verifies `getProviderLogoAndName("zai").displayName === Providers.ZAI` (the regression — previously the raw value `"zai"` was returned as the display name because no mapping existed), one verifies the placeholder.

**Why it matters.** The backend was already complete (`/public/providers` returns `zai`, docs page exists), so users could *create* a Z.AI model via the API and see it work, but the UI dropdown silently omitted Z.AI from new-model creation. Pure UI parity gap, references issue #25482.

**Concerns.**
1. **The fix is exactly the minimum surface required and matches the established pattern** for every other provider in the file — enum, map entry, placeholder. No risk.
2. **`displayName` chosen as `"Z.AI (Zhipu AI)"`** disambiguates the brand. Good choice; matches the docs page title.
3. **No logo entry added to `providerLogoMap`.** Skim of the diff doesn't show one; if there's an asset, add it; if not, the existing fallback (default-provider logo) is fine. The test only asserts `displayName`, not `logo`, so absence is intentional.
4. **Placeholder `"zai/glm-4.5"`** is reasonable; the docs reference `glm-4.5` as the flagship. If `glm-4.6` or similar ships before this merges, easy follow-up.
5. **Tests reference issue link** in comments (line 70) — good provenance for future archaeologists.

Trivially correct. Ship.
