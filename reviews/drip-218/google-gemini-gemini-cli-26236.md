---
pr-url: https://github.com/google-gemini/gemini-cli/pull/26236
sha: 8fba44dfc0f9
verdict: merge-after-nits
---

# fix(bot): productivity and backlog optimizations

Multi-layer fix to the gemini-cli CI bot covering trigger surface, comment-API fallback, prompt scope discipline, and PR-creation resilience. The trigger gate at `.github/workflows/gemini-cli-bot-brain.yml:41-47` adds a sibling `pull_request_review_comment` arm with the same author-association allowlist (`COLLABORATOR/MEMBER/OWNER`) and bot-self-trigger guard, so review-thread `@gemini-cli` mentions now actually run the workflow — previously the bot was deaf on PR review threads, which is where most code-review back-and-forth happens.

The comment-fetch fix at `:122` is small but load-bearing: `gh api repos/.../issues/comments/$ID -q '.body' >> trigger_context.md 2>/dev/null || gh api repos/.../pulls/comments/$ID -q '.body' >> trigger_context.md` falls back to the pulls-comments endpoint when the issues-comments endpoint 404s, because review-thread comments live under `/pulls/comments/` not `/issues/comments/` despite sharing the comment ID space. The same `pull_request_review_comment` event-name guard is fanned out to four downstream `if:` conditions (`Run Critique Phase`, `Generate Patch`, `Generate GitHub App Token`, `Create or Update PR`).

The PR-creation hardening at `:248-280` adds three real failures to handle: (a) `git push` fails → retry with `FALLBACK_PAT` env (`secrets.GEMINI_CLI_ROBOT_GITHUB_PAT`) by rewriting the `origin` remote URL inline; (b) PR exists but is `CLOSED` → cut a new branch named `${BRANCH_NAME}-retry-${run_id}` and open a fresh draft; (c) the `git remote set-url origin "https://x-access-token:${FALLBACK_PAT}@..."` shape uses GitHub's documented PAT-in-URL auth pattern.

The `critique.md:92-97` "Strict Scope Constraint" addition is the right prompt-engineering shape — explicit FORBIDDEN + the causal "do not attempt to complete pending tasks from the memory ledger" — but it does not address the orthogonal problem of the critique phase staging files outside `git diff --staged` via *new* `git add` calls; the constraint says "ONLY critique and fix the files explicitly included in `git diff --staged`" but the very next step is "Re-stage the file with `git add`" which the model could interpret as license to stage anything.

Nits: (1) the `permission-contents`/`permission-pull-requests`/`permission-issues` keys at `:201-203` replaced the prior single-line `permissions: '{"contents": "write", ...}'` JSON blob — this matches the newer `actions/create-github-app-token@v2` API but drops the `workflows: write` permission silently, which will break any future bot patch that touches `.github/workflows/`; (2) `FALLBACK_PAT` in URL is functional but the documented best practice is git credential helpers — the URL form will leak in `set -x` debug logs; (3) the `--retry-${run_id}` branch naming will accumulate stale branches on every retry — needs a cleanup workflow.

## what I learned
The right shape for "bot got partially deaf on a new event surface" fixes is to fan the event-name guard out to *every* downstream `if:` condition in lockstep — missing one creates a half-working bot that triggers but doesn't post, which is worse UX than fully-deaf because the user assumes their request was ignored intentionally.
