# sst/opencode PR #25044 — fix(skill): require clear skill matches

- PR: https://github.com/sst/opencode/pull/25044
- Head SHA: `0398f4b5087ea4442088f61bfd7e48e480c4311d`
- Files touched: `packages/opencode/src/session/system.ts`, `packages/opencode/src/tool/registry.ts`, `packages/opencode/src/tool/skill.txt`, `packages/opencode/test/session/system.test.ts`, `packages/opencode/test/tool/skill.test.ts` (+114/-6, 5 files)

## Specific citations

- System-prompt guidance changes at `system.ts:71-74`: the previous one-liner "Use the skill tool to load a skill when a task matches its description" is replaced with three explicit gating clauses — (a) "Use the skill tool when the user explicitly asks for a skill, or when the task clearly matches the skill's named domain and description", (b) "For repository- or domain-specific skills, the task must be about that repository or domain", (c) "Do not load a skill based only on generic workflow overlap, command names, or tool names in the skill description".
- Mirrored at `registry.ts:251-260` (the tool description block) and again at `skill.txt:1-7` (the in-tool docstring). All three sites carry the same three clauses verbatim — important because the two surfaces (system-prompt + tool-description) are both ingested by the model and drift between them is a known overfit hazard.
- New regression at `system.test.ts:69-119`: writes a synthetic `example-repo-dev-loop` `SKILL.md` with a description that overlaps generic dev-loop language ("Implement bug fixes ... separate worktree from origin/main, run local validation plus repeated codex review loops, commit with repo conventions, publish a ready PR ..."), then asserts all three guidance clauses are present in the system-prompt output and that the skill name + a description fragment are emitted. Locks the prose verbatim — drift between `system.ts` and `registry.ts` will fail this test.
- Companion test in `skill.test.ts:13` adds `ModelID, ProviderID` imports — likely the same `as any` → typed-constructor cleanup pattern used in #25053.

## Verdict: merge-after-nits

## Concerns / nits

1. **Soft contract / prompt-only fix**: same class as gemini-cli #26230 from drip-193 (the `exit_plan_mode` shell-escape fix). Stronger guardrail would be a runtime check at the skill dispatcher: when the matched skill's name is repository-scoped (e.g., contains a repo identifier) and the active workspace doesn't match, log a warning or refuse the load. The prompt clause is a hint to the model; the runtime check would catch the bug class even when prompt drift erodes the guidance over future model upgrades. Worth a follow-up.
2. **Verbatim prose lock in the regression**: `expect(output).toContain("Use the skill tool when the user explicitly asks for a skill, or when the task clearly matches the skill's named domain and description.")` at `system.test.ts:96` is a string-equality assertion on guidance prose. Any future tightening of the wording will break the test even when the semantic intent is preserved. Consider asserting on stable substrings like `"explicitly asks for a skill"` and `"named domain and description"` separately rather than the full sentence — same coverage, lower friction for prose tweaks.
3. **No negative test**: the regression only checks that the new clauses are *present* in the output. A complementary assertion that the old single-clause sentence ("Use the skill tool to load a skill when a task matches its description.") is *absent* would lock the replacement (vs accumulation) intent. Otherwise a future edit that adds new prose without retiring the loose guidance silently degrades back.
4. **Three-site duplication**: the same three clauses now live verbatim in `system.ts`, `registry.ts`, and `skill.txt`. Worth extracting to a `SKILL_GUIDANCE_CLAUSES` constant in a shared module so the three surfaces can't drift; the regression already locks `system.ts` but doesn't lock `registry.ts` or `skill.txt`.
5. The synthetic skill description in the test at `:75-80` is well-chosen — it has *exactly* the kind of generic-workflow-overlap shape that the new clauses target ("dev loop", "review loops", "PR", "validation"). A second test case that asserts a *legitimate* skill match still works (e.g., a `MyRepoDev` skill loaded for a task about MyRepo) would prevent the gate from accidentally hardening into an over-rejection.
