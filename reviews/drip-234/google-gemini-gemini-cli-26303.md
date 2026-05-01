# google-gemini/gemini-cli#26303 — feat(bot): enforce evaluation role and multi-iteration feedback loop

- **PR**: https://github.com/google-gemini/gemini-cli/pull/26303
- **Head SHA**: `c6121d51134ab83c34a6757edca88f33159c7dbd`
- **Size**: +124 / -75, 4 files
- **Verdict**: **needs-discussion**

## Context

The `gemini-cli-bot` automation has two LLM-driven phases: a "Brain" agent that proposes changes (stages files via `git add`) and a "Critique" agent that reviews the staged diff and emits `[APPROVED]` or `[REJECTED]`. Two real problems with the prior shape:

1. **Critique was allowed to apply fixes.** The prior `critique.md` told Critique to *also* edit and re-stage files. This conflated evaluation with implementation — same prompt-author surface that approves the change can also silently rewrite it.
2. **Single-shot.** If Critique rejected, the run failed; there was no opportunity for the Brain to react to the feedback and try again.

This PR converts to a single workflow step (`Run Brain and Critique Loop`) that runs up to `MAX_ITERATIONS=2` Brain → Critique cycles, with rejection feedback piped back into the Brain prompt for the next iteration.

## Design analysis

### Workflow shape (`gemini-cli-bot-brain.yml:148-220`)

```bash
MAX_ITERATIONS=2
ITERATION=1
while [ $ITERATION -le $MAX_ITERATIONS ]; do
    # BRAIN PHASE
    cat trigger_context.md > combined_prompt.md
    if [ -f "critique_feedback.md" ]; then
       cat critique_feedback.md >> combined_prompt.md
    fi
    cat "$PROMPT_PATH" tools/gemini-cli-bot/brain/common.md >> combined_prompt.md
    node bundle/gemini.js --policy ... -p "$(cat combined_prompt.md)"
    # ...fallback message if no comment generated...

    # CRITIQUE PHASE (gated on enable_prs / issue_comment / pull_request_review_comment / run_interactive)
    if [ "${{ ... gate ... }}" != "true" ]; then
        echo "[APPROVED]" > critique_result.txt; break
    fi
    if git diff --staged --quiet && [ ! -s "issue-comment.md" ] && [ ! -s "pr-comment.md" ]; then
       echo "[APPROVED]" > critique_result.txt; break
    fi
    node bundle/gemini.js --policy ... -p "$(cat tools/gemini-cli-bot/brain/critique.md)" 2>&1 | tee critique_output.log
    if [ "${PIPESTATUS[0]}" -eq 0 ] && grep -q "\[APPROVED\]" critique_output.log && ! grep -q "\[REJECTED\]" critique_output.log; then
       echo "[APPROVED]" > critique_result.txt; break
    else
       if [ $ITERATION -lt $MAX_ITERATIONS ]; then
         # Build feedback file, reset working tree, retry
         echo "<critique_feedback>" > critique_feedback.md
         echo "# Critique Feedback (Iteration $ITERATION)" >> critique_feedback.md
         cat critique_output.log >> critique_feedback.md
         echo "</critique_feedback>" >> critique_feedback.md
         git reset; git checkout .
         rm -f pr-description.md branch-name.txt pr-comment.md pr-number.txt issue-comment.md bot-changes.patch
       else
         echo "[REJECTED]" > critique_result.txt
         git diff --staged > bot-changes.patch || true
         break
       fi
    fi
    ITERATION=$((ITERATION+1))
done
```

The `git reset; git checkout .` between iterations is the load-bearing isolation move — without it, iteration 2's Brain would see iteration 1's partially-rejected state on disk and likely build on top of it. The explicit `rm -f` of generated artifacts (pr-description, branch-name, pr-comment, pr-number, issue-comment, bot-changes.patch) prevents iteration 2 from being misled by stale generated files. ✓

### Critique role (`brain/critique.md:5-12`)

Old: "You are responsible for applying fixes to the scripts if you detect any issues..."  
New: "You are an evaluator ONLY. You MUST NOT apply fixes or modify the code yourself."

The "Implementation Mandate" section is replaced by an "Evaluation Mandate" that explicitly says "Do NOT edit the code yourself." Removed `git add` instructions throughout. This is the right separation-of-concerns split.

### New `Systemic Simulation` (`brain/critique.md:90-99`)

> If the modified scripts or workflows involve time-based triggers (e.g., cron schedules), grace periods, or staleness checks:
> - You MUST explicitly write out a timeline simulation in your response.
> - Step through the execution day by day (e.g., Day 1, Day 7, Day 14).
> - Ensure that the execution frequency (the cron schedule) aligns perfectly with the logical grace periods promised in the code or comments.

This addresses a real prior failure mode (referenced indirectly by the new checklist item #12 "Architectural Conflict") where stale-issue policies and cron schedules went out of sync silently.

### New checklist item #12 (`brain/critique.md:73-77`)

> Architectural Conflict: Does this change tune a system while ignoring a conflicting system in the repository? You must `[REJECT]` changes that only treat the symptom of an architectural conflict. However, ensure the systems are actually conflicting (contradictory behavior) and not just complementary before demanding consolidation.

Plus a parallel requirement added to `brain/metrics.md:80-87` ("Identify Architectural Overlap") so the Brain proactively looks before the Critique catches it.

### `brain/common.md:99-105`

Adds a sentence: "removing duplicated, conflicting, or obsolete legacy workflows is considered the ultimate 'surgical' fix. Do not hesitate to delete files or workflows if your evidence shows they are conflicting with standard practices."

## What's right

- **Role separation.** Critique-as-evaluator-only is correct — same agent shouldn't both judge and rewrite. Reduces silent rewrites that bypass the Brain's discipline.
- **Iteration with feedback piping.** Single-shot was wasteful when the rejection was small/fixable. The `<critique_feedback>` block format keeps the feedback structurally distinct from the rest of the prompt.
- **Per-iteration working-tree reset.** Mandatory for honest iterations.
- **Cron-vs-grace timeline simulation.** Real prior failure mode addressed at the checklist level.
- **Combined-step env carryover.** Merging Brain + Critique into one `Run Brain and Critique Loop` step means env vars (`GEMINI_API_KEY`, `GITHUB_TOKEN`, `GEMINI_MODEL`) are set once. The prior split required redeclaring on the second step.

## Discussion blockers

1. **`MAX_ITERATIONS=2` is borderline.** With only 2 iterations, the Brain gets exactly one chance to react to feedback. For changes that the Critique flags on multiple axes (security + timeline + architectural), one revision pass is unlikely to clear all three. A `MAX_ITERATIONS=3` or workflow-input-configurable cap would give more headroom. Counter-argument: iterations cost LLM calls and CI time; 2 may be the right cost ceiling. Worth a PR-body justification of why 2 specifically.

2. **Feedback-loop drift risk.** The Brain prompt now contains a `<critique_feedback>` block from a *prior* run. If the Critique was wrong (false positive on architectural conflict, e.g.), the Brain is now biased toward "fixing" something that wasn't broken, potentially introducing regressions to satisfy a hallucinated objection. The PR has no escape valve — the Brain must address the feedback, no path to "Critique was wrong, ignore it." Consider adding: if iteration-2 Brain decides the iteration-1 critique feedback was incorrect, it should be allowed to write that justification into a `brain_response_to_critique.md` and the Critique on iteration 2 should weigh that. Right now there's no such channel.

3. **`git diff --staged --quiet && [ ! -s "issue-comment.md" ] && [ ! -s "pr-comment.md" ]` short-circuit at `:79`.** The condition "no staged changes AND no comments" → auto-approve and break. This is correct for "Brain decided no action is needed" but **also** approves the case where Brain *failed silently* and produced nothing. The fallback-message branch at `:65-68` writes `issue-comment.md` if `TRIGGER_ISSUE_NUMBER` is set and no comments were generated, which catches issue/PR triggers — but for *scheduled* triggers with no `TRIGGER_ISSUE_NUMBER`, the silent-failure case auto-approves with no output. Worth either always writing a fallback log or distinguishing "no action needed" (explicit Brain decision) from "Brain produced nothing" (silent failure).

4. **Critique `[REJECTED]` detection is purely substring.** `grep -q "\[REJECTED\]" critique_output.log` — if the Critique discusses a file containing the literal text `[REJECTED]` (e.g. a previous lessons-learned entry), it auto-rejects. Inverse for `[APPROVED]`. Consider anchoring to end-of-output or requiring a specific format like `## Final Verdict\n[APPROVED]`. The Critique prompt does say "at the very end of your response" but enforcement is grep, not parser.

5. **The `# Critique Feedback (Iteration $ITERATION)` heading uses `Iteration $ITERATION` which is the current iteration index.** On iteration 2, the feedback was generated *during* iteration 1's critique — heading should be `Iteration 1` (the iteration that produced this feedback) not the current iteration. Off-by-one in the audit trail. Cosmetic but worth fixing.

6. **No artifact upload on success path.** When iteration-1 succeeds, the rejected iteration-1 prep work is gone (well, there was none to reject). When iteration-2 succeeds after iteration-1 rejection, the iteration-1 critique log and feedback file are not preserved as artifacts — debugging "why did the bot need 2 iterations to land this PR" would benefit from those.

7. **`ci-policy.toml` is shared by both Brain and Critique.** The Critique role-change makes Critique an evaluator-only role, but it still runs under `--policy tools/gemini-cli-bot/ci-policy.toml` which presumably allows file edit tools. Even though the prompt says "do not edit", a tighter policy that *physically forbids* `write_file` / `git_add` from the Critique side would prevent prompt-injection-driven critique-rewrites that bypass the new prompt-level rule. The new PR explicitly mentions prompt-injection scanning at checklist item #13 — applying the same defense-in-depth thinking to Critique's tool surface seems consistent.

## Verdict

**needs-discussion** — directionally correct (role separation + iterative feedback loop + timeline simulation + architectural-overlap awareness), but with several auditability/safety items that warrant maintainer sign-off before landing: `MAX_ITERATIONS` justification, no-escape-valve from a wrong critique, silent-failure auto-approve case, grep-based verdict parsing, the `Iteration $ITERATION` heading off-by-one, missing artifact upload on iterated runs, and the policy-tier defense-in-depth question for Critique's tool surface.
