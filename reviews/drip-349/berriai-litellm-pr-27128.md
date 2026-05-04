# BerriAI/litellm#27128 — fix(ui): include unified access groups in Add Model dropdown

- **URL**: https://github.com/BerriAI/litellm/pull/27128
- **Head SHA**: `f969eb8c4884`
- **Diffstat**: +112 / -12
- **Verdict**: `merge-after-nits`

## Summary

Wires the unified `useAccessGroups()` hook (backed by `LiteLLM_AccessGroupTable` / `/v1/access_group`) into the Add Model and Edit Auto-Router surfaces so the "Model Access Group" picker shows both legacy `model_info.access_groups` entries and admin-managed unified groups. Adds a Vitest case asserting the merged dropdown contents.

## Findings

- `ui/litellm-dashboard/src/app/(dashboard)/models-and-endpoints/ModelsAndEndpointsView.tsx:96-118` — `availableModelAccessGroups` now unions legacy and unified sources via a `Set`, then `.sort()`s. Good: dedupes case-sensitively, stable order. Consider `.toLocaleLowerCase()` keying inside the set if you want `Engineering` and `engineering` to collapse — current behavior treats them as distinct, which may surprise admins.
- `ui/litellm-dashboard/src/components/add_model/AddModelForm.test.tsx:104-126` — `useAccessGroups` is mocked to return one row (`engineering`); the new test then asserts both legacy `model-group-1`/`model-group-2` and the unified `engineering` are visible. Coverage is exactly what's needed.
- `ui/litellm-dashboard/src/components/add_model/AddModelForm.test.tsx:298-321` — uses `act(async () => { await userEvent.click(...) })`. With `@testing-library/user-event` v14+, `userEvent.click` already handles act wrapping; the explicit `act` is redundant and triggers a console warning in some setups. Drop it or keep only one layer.
- The early-return guard `if (!modelDataResponse?.data && !unifiedAccessGroupsData) return [];` correctly handles the "neither loaded" case while letting either source populate alone.

## Recommendation

Behavior is right. Address the case-folding decision (document or implement) and trim the redundant `act` wrap, then merge.
