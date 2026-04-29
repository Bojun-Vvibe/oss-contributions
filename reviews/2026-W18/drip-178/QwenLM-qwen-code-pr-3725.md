# QwenLM/qwen-code#3725 — chore: remove legacy Gemini workflows

- **Repo:** QwenLM/qwen-code
- **PR:** #3725
- **Head SHA:** 2c48f3446b70f711506a492e7922f9a5754b1808
- **Base:** main
- **Author:** community (pomelo-nwu)

## Summary

Pure deletion PR removing six GitHub Actions workflow files that were inherited from the upstream `google-gemini/gemini-cli` fork and still hardcoded `if: github.repository == 'google-gemini/gemini-cli'` guards (so they were dead-on-arrival in the qwen-code fork). 0 additions, 736 deletions across six `.github/workflows/*.yml` files: `community-report.yml`, `eval.yml`, `gemini-automated-issue-dedup.yml`, `gemini-scheduled-issue-dedup.yml`, `gemini-self-assign-issue.yml`, and `no-response.yml`. All six referenced fork-specific secrets (`APP_ID`, `PRIVATE_KEY`), Googler-membership API checks (`gh api orgs/googlers/members/...`), or fork-specific labels.

## File:line references

- `.github/workflows/community-report.yml` (deleted, was 197 lines) — used `gh api orgs/googlers/members/${author}` to classify contributor as Googler vs community; useless in qwen-code
- `.github/workflows/eval.yml` (deleted) — fork-specific evaluation pipeline
- `.github/workflows/gemini-automated-issue-dedup.yml` (deleted) — used a Gemini-CLI-specific container
- `.github/workflows/gemini-scheduled-issue-dedup.yml` (deleted) — scheduled variant of above
- `.github/workflows/gemini-self-assign-issue.yml` (deleted) — fork-specific self-assignment automation
- `.github/workflows/no-response.yml` (deleted) — stale-issue closer with fork-specific label config
- All six guarded by `if: github.repository == 'google-gemini/gemini-cli'`, so behavior change to the qwen-code fork is **zero**

## Verdict: **merge-as-is**

## Rationale

Cleanest possible deletion PR: every workflow being removed is gated on `github.repository == 'google-gemini/gemini-cli'`, which is statically false in `QwenLM/qwen-code`, so the removal cannot change observable CI behavior in this repo. The PR body's "Reviewer focus" question ("verify that no active Qwen Code release, CI, or issue-management process still depends on these removed workflows") is conservatively answered by reading each deleted file's `if:` guard at the top of its `jobs:` block — none could have been doing useful work in this fork.

Two extremely minor follow-ups, both non-blocking:

1. The PR body's testing matrix shows `npm run` as ⚠️ on Windows/Linux (untested). For a workflow-deletion PR this is fine — the deleted files have zero runtime effect on local builds — but consider adding a one-line note "no local build impact since deleted files only affect GitHub Actions" so future archaeologists don't wonder why Linux/Windows weren't validated.
2. Worth filing a follow-up to scrub for any **other** lingering `google-gemini/gemini-cli` repository-name guards in still-present workflow files (`grep -r "google-gemini/gemini-cli" .github/`) — those would also be dead code in the qwen-code fork and the same logic applies.

Net: ship it. The blast radius is provably zero, the diff is fork hygiene, and any deferred cleanup belongs in a follow-up PR.
