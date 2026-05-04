# block/goose #8982 — fold UI refactor review into code review skill

- SHA: `408c4b4981cd93890a2d37bb610ad09589910b21`
- State: OPEN, +42/-5 in `.agents/skills/code-review/SKILL.md` only

## Summary

Documentation-only PR that folds a previously-removed `ui-refactor-review` skill's unique guidance into the existing `code-review` skill. Adds a dedicated "UI Refactor Quality" section, sharpens several existing checklist items (component/function/file size thresholds, named/shared types, semantic HTML, stable i18n keys, redundant data, coverage drift), and adds a "Pass 4" to the silent self-review and an explicit instruction to omit "Applied Well" sections.

## Notes

- `.agents/skills/code-review/SKILL.md:13-14` — adds "Default to reporting what needs to be fixed. Do not include an 'Applied Well' or praise section unless the user explicitly asks for positive feedback." This is reinforced again at line 218. Good — single source of truth, but consider keeping just one instance to avoid drift if the policy is later softened. The reinforcement at 218 is the natural place since it's adjacent to the output-shape rules.
- `.agents/skills/code-review/SKILL.md:122-138` — new "UI Refactor Quality" section is well-scoped: explicitly gated on `ui/goose2` changes, calls out the SDK -> ACP -> goose path, and forbids ad-hoc `fetch()`/`invoke()` proxies for core behavior. Two questions: (1) the gating phrase "for `ui/goose2` changes" is repeated at lines 122 and 170 — if any other UI surfaces exist or arrive (e.g., a new product UI), the rule is silent. Worth a "any equivalent UI surface" hedge. (2) "Boundary Discipline" at line 130 is the most prescriptive line in the entire skill; if there's a legitimate exception (e.g., legacy `invoke()` call still needed), the skill currently gives the reviewer no escape hatch beyond ignoring its own rule.
- `.agents/skills/code-review/SKILL.md:55-58` — "Component Size: Treat these as smell thresholds, not hard limits: components around 200 lines, functions around 40 lines, files around 300 lines, JSX nesting around 4 levels." Useful operational guidance. Worth considering whether these numbers should be consistent with the broader Goose engineering-handbook if one exists, otherwise this becomes the de facto standard by accident.
- `.agents/skills/code-review/SKILL.md:91-94` — "Stable Extraction" + "Post-PR Shape" + "Partial Cleanup" are three subtle additions that together prevent a common reviewer failure mode (declaring victory because the PR moved the smell rather than removed it). Good.
- `.agents/skills/code-review/SKILL.md:170` — "Step 1, item 4: For ui/goose2 refactors, run the UI Refactor Quality pass before finalizing findings" is the operational hook that ties the new section to the workflow. Without this, the new content would be inert. Correct placement.
- `.agents/skills/code-review/SKILL.md:172` — Step 1 item 2 now reads "do not flag issues in unchanged code, but follow changed code paths into surrounding modules when needed to verify the issue." Good softening — the previous strict rule blocked legitimate cross-module verification.
- No tests; the file is a Markdown skill instruction. The PR body acknowledges "Verification: Not run; Markdown-only skill instruction change." That's appropriate, but a sample dry-run output (paste of running the updated skill on a known PR) would have been a meaningful smoke test.

## Verdict

`merge-as-is` — coherent consolidation of UI refactor guidance into the canonical code-review skill, with the right gating hooks and self-review additions. The two minor concerns (single source of truth for the no-praise rule; escape hatch for the SDK-only boundary rule) are stylistic and can be deferred. Pure docs change with low blast radius.
