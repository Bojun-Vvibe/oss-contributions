# Review: google-gemini/gemini-cli PR #26303

- **Title:** feat(bot): enforce evaluation role and multi-iteration feedback loop
- **Author:** gundermanc (Christian Gunderman)
- **Head SHA:** `1b021bddab8e76a2663bcbd6d24c7e40c8e776a4`
- **Verdict:** merge-after-nits

## Summary

Restructures the gemini-cli-bot CI workflow to (a) enforce that the
critique agent is evaluator-only (no longer allowed to apply fixes
itself) and (b) wrap brain + critique in a 4-iteration feedback loop:
on rejection, the workflow stages the critique output as feedback,
discards the changes, and re-invokes brain. Also adds `bottlenecks.ts`
and `priority_distribution.ts` metric scripts and tightens
`throughput.ts`. Critique prompt and `common.md` are updated to
reflect the new role boundaries.

## Specific-line comments

- `.github/workflows/gemini-cli-bot-brain.yml:155-219` — the
  `MAX_ITERATIONS=4` loop is the central change. Concerns:
  - The reset-on-rejection sequence is `git reset && git checkout . &&
    rm -f pr-description.md branch-name.txt pr-comment.md
    pr-number.txt issue-comment.md bot-changes.patch
    rejected-changes.patch`. `git checkout .` only restores tracked
    files; **untracked** files written by the brain (e.g., a brand new
    workflow file) will survive into the next iteration. Use
    `git clean -fd` (scoped, e.g., to `tools/` and `.github/`) in
    addition, or the next brain pass starts from a polluted tree.
  - `git reset` with no args unstages but keeps working-tree changes.
    Combined with `git checkout .`, that resets tracked-file edits but
    again misses untracked. Same point.
  - The whole loop body is inline shell inside YAML; at 60+ lines it
    is past the threshold where a small `tools/gemini-cli-bot/loop.sh`
    would be more reviewable and unit-testable.
- `.github/workflows/gemini-cli-bot-brain.yml:243` — adding
  `rejected-changes.patch` to the artifact upload is good for
  debuggability; ensure the artifact retention policy does not exceed
  what the existing one allows (no change is visible here, just worth
  confirming centrally).
- `tools/gemini-cli-bot/brain/critique.md` (60/60 diff, near full
  rewrite) — switching the critique role from "fix issues yourself" to
  "evaluator only, output `[APPROVED]` or `[REJECTED]`" is a clean
  separation of concerns and the right direction. Make sure the prompt
  explicitly forbids `git add` / `write_file`, not just discourages
  them, since the previous version *required* them.
- `tools/gemini-cli-bot/brain/common.md:88-104` — the new "Defensive
  Scripting & Resilience (MANDATORY)" section is a good addition. The
  "Preserve Exemptions" rule directly addresses real failures (stale
  bots dropping `security` / `pinned` exemptions). Worth referencing
  this section by name from the critique prompt's checklist so the
  evaluator knows to look for it.
- `tools/gemini-cli-bot/metrics/scripts/bottlenecks.ts` (101 new lines)
  and `priority_distribution.ts` (90 new lines) — net-new metric
  scripts. Without seeing them in full I can only flag that the
  pattern in sibling scripts (e.g., `backlog_health.ts` from #26302)
  uses inline `gh api graphql` interpolation of `$GITHUB_OWNER` —
  watch for shell-quoting bugs if any user-controlled input flows in.

## Risks / nits

- 4 iterations × (brain + critique) is a real cost increase per CI
  run. The loop should log iteration-level token spend so the team can
  decide if `MAX_ITERATIONS=4` is the right cap.
- The `git reset && git checkout .` cleanup leaves untracked files
  behind — meaningful given that the brain *creates* new files as its
  primary output.
- The `if: ... != "true"` test inside the bash block re-evaluates the
  same conditional that gates the surrounding step; consider moving
  the gate to step-level `if:` to keep the loop body simpler.

## Verdict justification

Real architectural improvement (evaluator role, feedback loop) with
sensible artifact debugging. The untracked-file cleanup gap is a real
correctness issue for the iteration loop and is the main reason this
is not "merge-as-is". **merge-after-nits.**
