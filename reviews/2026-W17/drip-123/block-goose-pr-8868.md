# block/goose#8868 — redesign skills library

- **PR**: https://github.com/block/goose/pull/8868
- **Author**: morgmart
- **Head SHA**: `3f6fb45b38dd82787ea5e3b7007378c4683ee097`
- **Verdict**: `merge-after-nits`

## Context

`goose` ships a skills system (markdown-formatted reusable instruction packs) discovered from a chain of project-scoped and home-scoped directories. Historically the project-scoped writable location was `<project>/.goose/skills` — a goose-specific path that doesn't compose with other agent ecosystems (Claude / `.claude/skills`, the broader `.agents/skills` convention used by spec-kitty and other tooling). This PR migrates the canonical project-scoped writable location to `<project>/.agents/skills`, makes it the *highest-priority* project source so it shadows the legacy `.goose/skills` on collision, and ships a UI redesign of the SkillsView that adds project-aware skill hydration plus a category/source filter chip set. The diff visible at lines 1-200 covers the Rust-side discovery + sources changes; the UI side is referenced but not in the visible window.

## What it changes

**Rust core**: `crates/goose/src/skills/mod.rs:8-13` rewrites `project_skills_dir` to return `<project>/.agents/skills` instead of `<project>/.goose/skills`. The `all_skill_dirs` walker at `:20-25` reorders the project-scoped directory list so `.agents/skills` is searched first, then `.goose/skills`, then `.claude/skills` — meaning a skill named `shared-skill` defined in both `.agents/skills/shared-skill/` and `.goose/skills/shared-skill/` resolves to the `.agents/skills` version. **Sources tests**: existing tests at `crates/goose/src/sources.rs:380` and `:541` are migrated from `.goose/skills` to `.agents/skills` paths. Two new tests pin the new behavior: `list_sources_reads_project_agents_skills` at `:54-73` proves the `.agents/skills` path is discovered with the right description, and `project_sources_prefer_agents_directory_over_legacy_goose` at `:75-113` constructs a collision (same skill name in both `.agents/skills` and `.goose/skills`) and asserts the listed result has length 1, points to `.agents/skills`, has the "preferred" description (not "legacy"), and the exported source contains the agents-directory content. **UI**: `ui/goose2/scripts/check-file-sizes.mjs:83-87` adds an exception entry for `SkillsView.tsx` raising the limit to 620 lines with a justification naming the centralized list/detail state, project-aware hydration, filter UX, and import/export flows. `ui/goose2/src/features/agents/ui/PersonaDetails.tsx:7` imports the new `DetailField` shared component.

## Design analysis

The discovery-order rewrite at `mod.rs:20-25` is the load-bearing decision and it's the right one. Putting `.agents/skills` *first* in the project-scoped list at `:21` means new skills written via the redesigned UI (which I assume targets the new canonical path) shadow any legacy `.goose/skills` entry with the same name. That's the correct migration semantic — users who copy their old skills over to `.agents/skills` get the new version, users who don't see no behavior change because `.goose/skills` is still discovered. The collision test at `:75-113` is the strongest piece of the diff: it proves the precedence contract end-to-end through the public `list_sources` API plus `export_source`, not just through the directory-walker function in isolation. That means a future refactor that accidentally swaps `.agents/skills` and `.goose/skills` in the search order will fail this test, not just a unit-level walker test that someone might forget to update.

The `project_skills_dir` *write* path at `:11-13` correctly returns the new canonical path, so any UI flow that calls `project_skills_dir(...)` to compute "where do I write a new skill?" will write to `.agents/skills`. Combined with the discovery-order change, this means the system is internally consistent: writes go to `.agents/skills`, reads prefer `.agents/skills`, no skill is silently shadowed by an older copy of itself.

## Concerns / nits

1. **No migration story for legacy `.goose/skills` users** — the diff doesn't move existing `.goose/skills` content, doesn't print a deprecation warning when reading from `.goose/skills`, and doesn't document the migration in the PR description visible portion. Users with skills in `.goose/skills` will continue to see them work (because `.goose/skills` is still in the search order), but won't know the canonical path has moved. A startup-time deprecation log when `.goose/skills` is non-empty AND `.agents/skills` is empty (i.e., "you have legacy skills, please move them") would close the doc gap. Not blocking but worth mentioning.

2. **`SkillsView.tsx` file-size exception at 620 lines** is a code-smell escape hatch — the justification ("centralizes list/detail state, project-aware skill hydration, category/source filtering, import/export flows, and detail-page action wiring pending a later decomposition") names five distinct concerns in one component. The "pending a later decomposition" hedge suggests the author knows this — if there's an existing tracking issue for the decomposition, the justification should link it; if not, one should be filed. Not blocking but the file-size exception list is meant to be an exit ramp, not a parking lot.

3. **`PersonaDetails.tsx` import of `DetailField`** at `:7` is one of presumably many UI changes not visible in the lines 1-200 window — out-of-scope for this review but worth noting that the PR appears to bundle the skills-library redesign with broader shared-component refactors. A reviewer should verify the PR description names this scope explicitly.

4. **The legacy-search-order test name `project_sources_prefer_agents_directory_over_legacy_goose` is good** — it documents the intent inline. But the test only checks the case where both directories have the *same skill name*. A symmetric test for "skills in both directories with different names" (which should both appear in the listing) would close the only remaining ambiguity in the precedence contract: does `.goose/skills` get *suppressed* when `.agents/skills` exists, or merely *shadowed on name collision*? The diff implements the latter (correct), and a test that proves it would be reviewer-friendly.

## Risks

Medium. The discovery-order change is strictly additive for the read path (`.agents/skills` is *added* to the search order, not substituted for `.goose/skills`), so no user loses access to existing skills. The write path *does* change — any future skill-write flow that calls `project_skills_dir` will write to the new location. If there's any UI flow that hard-codes `.goose/skills` for skill-writing rather than going through `project_skills_dir`, this PR would split writes between two directories silently. The diff visible window doesn't cover the UI write side, so this is a verification step for the maintainer.

## Suggestions

Land after: (a) adding a startup-time deprecation log for non-empty `.goose/skills` directories (or at minimum a one-line CHANGELOG note about the canonical path move), (b) filing or linking a tracking issue for the `SkillsView.tsx` decomposition the file-size justification references, and (c) verifying no UI write path hard-codes `.goose/skills`.

## What I learned

The "introduce new canonical path, search-order-prefer it, leave legacy in the chain" migration shape is the right pattern for any directory-convention rename — strictly additive on reads, write-redirected on writes, no user-data motion required. The collision-precedence test at `:75-113` is the highest-value test in the diff because it pins the *precedence semantic*, not just the *path-walking mechanic* — that's the pattern to copy for any future "search order matters" refactor.
