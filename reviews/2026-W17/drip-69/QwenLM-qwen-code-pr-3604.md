# QwenLM/qwen-code #3604 — feat(skills): parallelize loading + add path-conditional activation

- **Author:** wenshao
- **Head SHA:** `e20247bafb5a4649efb2e700ea1db465641bbab6`
- **Size:** +871 / -66 (11 files, 3 commits)
- **URL:** https://github.com/QwenLM/qwen-code/pull/3604

## Summary

Three commits stacked on the new path-conditional skill activation
pipeline:

1. `99b9fe4de` — perf: replace nested serial `for-await` loops in
   `SkillManager` with `Promise.all` at all three discovery layers
   (level → provider dir → skill dir).
2. `5432f5ddc` — feat: skills with a `paths:` glob in frontmatter stay
   out of the model-facing `<available_skills>` listing until a tool
   call touches a matching path, at which point they activate for the
   rest of the session.
3. `160462344` — fix: two activation-pipeline bugs surfaced by
   `/ultrareview` (cross-level shadow leaks paths;
   `paths:` + `disable-model-invocation: true` self-contradiction).

## Specific findings

- **Parallelization (commit 1) is the right shape.**
  `packages/core/src/skills/skill-manager.ts` — fanout at every layer
  was the obvious win; the comment in `loadSkillsFromDir` explicitly
  notes that the `parseErrors` Map iteration order now reflects
  parse-finish order rather than on-disk order, and verifies the
  single consumer (`tools/skill.ts:339`) doesn't depend on order.
  That's the kind of side-effect callout that prevents a future
  consumer from quietly assuming a property the API never promised.

- **Activation registry (commit 2) is well-decomposed.**
  `packages/core/src/skills/skill-activation.ts` (new, +118) is a
  picomatch-based registry scoped to project root (refuses paths
  outside it). The model-facing surface in
  `packages/core/src/tools/skill.ts` adds a distinct
  `validateToolParams` error path for "gated by path-based
  activation" so the model can disambiguate *not found* from
  *registered but not yet active* — that's the right contract,
  because the failure mode otherwise looks indistinguishable from a
  typo. The activation hook lives next to the existing
  `ConditionalRulesRegistry` hook in `coreToolScheduler.ts`, so
  there's a single place where "tool call → registry side-effect"
  fires.

- **The two bug-3 fixes are exactly the kind of subtle defects a
  human reviewer wouldn't catch on first pass.** The PR body
  documents both:
  - **bug_001 (shadow-level leak):** Same skill name at two levels
    (project + extension) with different `paths:` globs. `listSkills()`
    dedupes by precedence (project wins), but the registry was being
    built from the raw concatenation, so a path matching only the
    *shadowed* (extension) glob still flipped the *visible* (project)
    skill to active. Real bug. Fix: dedupe before registry build.
  - **bug_004 (paths + disable-model-invocation):** The skill enters
    the registry, the activation `<system-reminder>` fires, then
    `SkillTool` refuses the call because the disabled flag had filtered
    the skill out of both `availableSkills` and
    `pendingConditionalSkillNames`. Model gets "skill is now available"
    immediately followed by "skill not found" — strictly worse than
    silent. Fix: drop `disableModelInvocation` skills *before*
    splitting on `paths`.

- **The dedup-key fix in commit 1 (review history item 1) is load-bearing.**
  `SkillTool.refreshSkills` was sourcing the model-invocable-commands
  dedup key from `availableSkills` (already filtered) rather than
  `allSkills`, so a path-gated skill could leak into
  `<available_commands>` and bypass the activation contract. Easy to
  miss; good catch.

- **Activation state is single-instance and not persisted across
  registry rebuilds.** Documented as a deliberate choice mirroring
  `ConditionalRulesRegistry`. That's defensible — a watcher-driven
  reload should reset activation rather than carrying stale state into
  a re-evaluated skill set — but it's worth a `<system-reminder>` text
  callout when a watcher rebuild wipes activations mid-session, so the
  model knows why a previously-available skill stopped being listed.
  Not blocking.

- **Test surface is genuinely large and matches the change shape.**
  286 new lines in `skill-manager.test.ts`, 133 in
  `skill-activation.test.ts`, 62 in `tools/skill.test.ts`, plus
  scheduler coverage. PR body claims 200 passed across 5 files for the
  skill-area subset; matrix: 9 CI jobs (mac/ubuntu/windows × Node
  20/22/24) green per the author.

- **One remaining nit, not a blocker.** The new
  `<system-reminder>` that fires on activation could mention *which
  path* triggered it (the registry has the matched glob, just not
  threaded to the emit site). Helps debugging when a glob is overly
  broad and a skill activates "for no apparent reason."

## Verdict

`merge-after-nits`

(Optional `<system-reminder>` enrichment with the matched path; the
core change is sound and unusually well-tested.)
