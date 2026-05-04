# QwenLM/qwen-code PR #3754 — feat(review): expand review pipeline + qwen review CLI subcommands

- **PR:** https://github.com/QwenLM/qwen-code/pull/3754
- **Author:** wenshao
- **Head SHA:** `124a3834` (full: `124a3834dd52a268feef0c77205980c49deca903`)
- **State:** MERGED
- **Files touched (selected):**
  - `docs/users/features/code-review.md` (+38 / -26)
  - `packages/cli/src/commands/review.ts` (+39 / -0, new entrypoint)
  - `packages/cli/src/commands/review/cleanup.ts` (+112 / -0, new)
  - `packages/cli/src/commands/review/deterministic.ts` (+733 / -0, new)
  - `packages/cli/src/commands/review/fetch-pr.ts` (+214 / -0, new)
  - `packages/cli/src/commands/review/lib/gh.ts` (+69 / -0)
  - `packages/cli/src/commands/review/lib/git.ts` (+44 / -0)
  - `packages/cli/src/commands/review/lib/paths.ts` (+43 / -0)
  - `packages/cli/src/commands/review/load-rules.ts` (+154 / -0)
  - `packages/cli/src/commands/review/pr-context.ts` (+317 / -0)

## Verdict

**merge-after-nits**

## Specific refs

- `docs/users/features/code-review.md:29-46` — the agent count grows from 5 to 9 (Correctness/Security split, dedicated Test Coverage agent, Undirected Audit becomes 3 personas 6a/6b/6c). The token-cost table at `code-review.md:256-264` correctly re-derives the bound: same-repo 11-13 calls, cross-repo 10-12. Math checks: 9 + 1 verify + (1-3) reverse audit = 11-13. Good.
- `docs/users/features/code-review.md:191` — the rule-injection note now says "agents (1-6)" but the build/test agent is now Agent 7 and explicitly excluded from rule injection. That's the right scoping (build/test runs commands, not LLM judgment) — worth one sentence saying so explicitly so a future contributor doesn't "helpfully" extend rule injection to Agent 7.
- New `packages/cli/src/commands/review/deterministic.ts` (+733 lines) is the largest single file added. Without diff body for that file, the size alone is a review-effort flag; a 700+-line single file is hard to PR-review and should ideally be split per-checker. Same observation for `pr-context.ts` (+317).
- The PR body documents two important guardrails worth pinning: (a) self-authored PRs auto-downgrade `APPROVE`/`REQUEST_CHANGES` to `COMMENT` to dodge GitHub HTTP 422; (b) `APPROVE` verdict is auto-downgraded to `COMMENT` if any check-run is failing or all are pending. Both are correct policies and the docs correctly call them out (`code-review.md:163-166`).

## Rationale

Substantive feature PR that moves a large chunk of bash/inline-shell logic out of `SKILL.md` and into typed TS subcommands under `packages/cli/src/commands/review/`. That direction is right — it makes the review pipeline testable, cross-platform (Windows-friendly), and hides the gnarly `gh` / `git` plumbing behind named commands. The two guardrails (self-PR downgrade, CI-red downgrade) are exactly the kind of policy-in-code that prevents the LLM from doing the wrong thing on GitHub. Nits before next iteration: (1) split `deterministic.ts` (733 lines) per linter/typechecker so each checker is independently reviewable and tested; (2) tighten the doc note about Agent 7 being exempt from rule injection; (3) the iterative reverse audit "1-3 rounds" hard cap should be a named constant so it's grep-able and trivially adjustable. None of these block the merge; this is a strong upgrade to the review pipeline shape.

