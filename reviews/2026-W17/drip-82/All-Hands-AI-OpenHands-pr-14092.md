# All-Hands-AI/OpenHands PR #14092 — chore(deps): bump dawidd6/action-download-artifact from 15 to 20

- Repo: All-Hands-AI/OpenHands
- PR: #14092
- Head SHA: `7fda1984`
- Author: @app/dependabot
- Diff: +1/-1 in `.github/workflows/pr-review-evaluation.yml`

## What changed

Single-line dependabot bump of `dawidd6/action-download-artifact` from `@v15` to `@v20` at `.github/workflows/pr-review-evaluation.yml:31`. Used inside the `pr-review-evaluation` workflow's "Download review trace artifact" step which downloads a trace produced by the `pr-review-by-openhands.yml` workflow.

## Specific observations

- The bump crosses five major versions. v16 reworked the `workflow` parameter resolution (now requires either `workflow` or `run-id` not both); v17 added `if-no-artifact-found: warn|fail|ignore` and changed the default from silently succeeding to `fail` — this matters because the call site at `.github/workflows/pr-review-evaluation.yml:30` sets `continue-on-error: true` but does not set `if-no-artifact-found`, so if the upstream evaluation workflow hasn't produced a trace, v20 will now fail the step (caught by `continue-on-error`) instead of silently no-op'ing. v18 added `allow_forks: true|false` defaulting to `false` for security; the workflow at the call site is `pull_request_target`-style (judging by the trace-pulling pattern), so confirm whether fork PRs need the flag enabled.
- The `with:` block at `:32-37` (per the surrounding diff context) only specifies `workflow:`, `pr:` (likely), and `name:` — none of the new v17/v18/v20 parameters. That's fine for the default behavior but means any silent default-flips in v17/v20 will land unannounced. Worth verifying against the action's CHANGELOG that no default was flipped between v15 and v20 for the parameters this workflow does use.
- Action is community-maintained (`dawidd6` is an individual, not a vendored org) — pin to a full SHA rather than a floating major tag is a stronger supply-chain posture; see how `actions/setup-python@v6` was bumped in #14093 also using a floating tag for org consistency, but a SHA pin would survive a hijacked tag.

## Verdict

`merge-after-nits`

## Rationale

Tiny dependabot PR but the version jump crosses behavior-default flips (notably `if-no-artifact-found` in v17). Pre-merge: explicitly set `if-no-artifact-found: warn` to preserve previous "silently succeed" semantics, kick the workflow once on a PR that has no trace artifact to confirm it doesn't start failing pipelines, and consider a SHA pin while you're touching the line.
