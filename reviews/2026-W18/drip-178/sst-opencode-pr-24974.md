# sst/opencode#24974 — tweak: adjust order of system prompt instructions: Global, Project, Skills

- **Repo:** sst/opencode
- **PR:** #24974
- **Head SHA:** f021293296c7ff3fe5e415d5c645d1c942244676
- **Base:** dev
- **Author:** community

## Summary

Re-orders instruction composition so the final system prompt array becomes `[env, ...instructions, skills?]` instead of `[env, skills?, ...instructions]`, AND re-orders the `instructions` accumulator inside `Instruction.layer` so global AGENTS.md/CLAUDE.md is added *before* project-level files. Net effect on the model: the system prompt now reads global → project → skills, which matches the mental model that more-specific guidance should appear later (LLMs weight later-in-context instructions more heavily).

## File:line references

- `packages/opencode/src/session/instruction.ts:125-130` — global-files block moved above project-files block
- `packages/opencode/src/session/instruction.ts:140-147` — project-files block (was previously above global)
- `packages/opencode/src/session/prompt.ts:1448` — final array changed from `[...env, ...(skills ? [skills] : []), ...instructions]` to `[...env, ...instructions, ...(skills ? [skills] : [])]`
- `packages/opencode/test/session/instruction.test.ts:254-259` — test rewritten from order-agnostic `toContain` to ordered `rules[0] === global`, `rules[1] === project`

## Verdict: **merge-after-nits**

## Rationale

The reordering rationale is sound and the test is now strictly stronger (asserts order, not just presence). Concerns:

1. **Behavior change is silent for users who already tuned their prompts to the old order.** A user who put a "do this last" override in `.opencode/AGENTS.md` (project-level, expecting it to win because it appeared after global) now gets it shadowed by skills. Add a one-line note to the PR body / CHANGELOG that the system-prompt instruction order changed from "global → skills → project" to "global → project → skills" so anyone debugging behavior drift after upgrade can correlate.
2. **Test coverage gap:** the test at `instruction.test.ts:254-259` only covers the (global + project) combo. Add a third assertion covering `(global + project + URL-loaded instruction)` to lock in that URL-loaded entries from `config.instructions` continue to land after both — otherwise the next refactor could silently re-shuffle URL-loaded items above project files.
3. The `break` at `instruction.ts:128` correctly preserves "first global match wins" parity with the project loop's existing `break` semantics — no concern, but worth a one-line code comment so the symmetry isn't accidentally lost.
4. Verify `prompt.ts:1448` doesn't have other call sites that build the `system` array independently — if there's a parallel path (e.g., a `compact` or `summarize` flow) that still uses the old order, the user will see inconsistent behavior between normal turns and compaction turns.
