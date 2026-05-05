# block/goose PR #9008 — remove skill categories

- Repo: `block/goose`
- PR: #9008
- Head SHA: `87e22199581b`
- Author: `morgmart`
- Scope: 11 files, +16 / -541
- Verdict: **merge-after-nits**

## What it does

Deletes the entire client-side skill-categorization layer in the `goose2` desktop UI. Skills are no longer bucketed into the heuristic `design / engineering / quality / research / writing / integrations / operations / productivity / general` taxonomy and the corresponding category dropdown / "Group by category" header is removed from the Skills view.

Concrete deletions:

- `ui/goose2/src/features/skills/lib/skillCategories.ts` — entire file (294 lines) gone, including the `SKILL_CATEGORY_ORDER` const tuple, the exported `SkillCategory` and `SkillViewInfo` types, the per-category slug allow-lists (`DESIGN_SLUGS`, `ENGINEERING_SLUGS`, `QUALITY_SLUGS`, `RESEARCH_SLUGS`, `WRITING_SLUGS`, `OPERATIONS_SLUGS`, `INTEGRATION_SLUGS`, `PRODUCTIVITY_SLUGS`), the `CATEGORY_KEYWORDS` keyword tables, `inferCategoryFromSlug`, `inferSkillCategory`, `withInferredSkillCategory`, and `derivePresentCategoriesInOrder`.
- `ui/goose2/src/features/skills/ui/SkillCategoryFilter.tsx` — entire file deleted (the multi-select category dropdown component).
- `ui/goose2/src/features/skills/lib/skillsHelpers.ts:80-95` (`filterSkills` signature) — `selectedCategories: SkillCategory[]` parameter removed; `filterSkills` now operates on raw `SkillInfo` and only does search + source/project filtering.
- `ui/goose2/src/features/skills/lib/skillsHelpers.ts:~360` (`groupSkills`) — drops category bucketing.
- `ui/goose2/src/features/skills/ui/SkillsView.tsx:~641-647` — `filterSkills(...)` callsite drops the category arg; whole `selectedCategories` state and `getCategoryLabel` plumbing removed.
- `ui/goose2/src/features/skills/ui/SkillsToolbar.tsx`, `SkillsListSections.tsx`, `SkillDetailPage.tsx` — drop `SkillViewInfo` / `SkillCategory` props and rendering.
- `ui/goose2/src/shared/i18n/locales/{en,es}/skills.json` — drop the `category*` strings.
- `ui/goose2/src/features/skills/ui/__tests__/SkillsView.test.tsx` and `tests/e2e/skills.spec.ts` — updated to drop category assertions.

## Why this is OK

- **Honest motivation.** PR body explicitly says "categories were based on client-side heuristics that could misclassify skills and implied product semantics that are not part of the skill protocol." That's correct — the categorization was a hand-maintained slug allow-list (`DESIGN_SLUGS = new Set(["adapt", "animate", "audit", "bolder", ...])`) plus a keyword bag-of-words match, neither of which scales to user-installed skills and neither of which has any backing in the MCP / Goose extension protocol.
- **Surgical scope.** Despite the -541/+16 footprint, every change traces back to deleting a self-contained module. No public protocol type touched, no DB migration, no extension API change. The `SkillInfo` type that downstream consumers care about is unchanged.
- **Tests updated in the same PR.** The `SkillsView.test.tsx` and `e2e/skills.spec.ts` updates land alongside the deletion, so CI guards the new behaviour.
- **No back-compat surface for the deleted types.** `SkillCategory` and `SkillViewInfo` were declared and consumed only inside `features/skills/`; they weren't exported through any package barrel that an external consumer could depend on. Safe to delete.

## Issues / nits

1. **No deprecation / settings-cleanup step.** If users had previously selected categories in the toolbar dropdown, that selection state is presumably persisted somewhere (Zustand, localStorage, or the desktop settings file). Worth a one-liner that purges any stored `selectedCategories` key on next load so dead state doesn't accumulate. If categories were never persisted, mention that in the PR body so reviewers don't have to grep.

2. **`groupSkills` semantics changed quietly.** Before: skills were grouped by category section in `SkillsListSections`. After: they're presumably grouped by source/project (or just listed flat). The PR description says "search plus source/project filtering" survives, but doesn't spell out the new grouping behavior. A screenshot would resolve this in 2 seconds.

3. **i18n key removal is hard-deletion.** Translators who maintain non-EN/ES locales (if any) will silently lose their `skills.category*` translations. Standard ICU practice is to leave the keys deprecated for one release. Low impact for goose2 if EN/ES are the only supported locales — confirm.

4. **The PR title "remove skill categories" is doing a lot of work.** Worth saying "remove **client-side inferred** skill categories from goose2 UI" since `block/goose` also has agent-side / extension-side concepts that aren't being touched.

5. **The 794-line goose UI test additions look correct.** No risk of leftover `inferCategory` callers — `grep -r "inferCategory\|SkillViewInfo\|SkillCategory" ui/goose2/src/` after this PR should return zero hits; quick sanity check before merging.

## Verdict
**merge-after-nits** — the deletion is well-motivated (heuristic taxonomy with no protocol backing), self-contained to the goose2 desktop UI, and ships with updated tests. The cleanup of any persisted `selectedCategories` state and a screenshot of the new flat/grouped view would make this fully ship-ready.
