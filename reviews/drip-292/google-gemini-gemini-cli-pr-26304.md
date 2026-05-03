# google-gemini/gemini-cli PR #26304 — Backlog Health & Stale Policy Optimization

- **Repo:** google-gemini/gemini-cli
- **PR:** #26304
- **Head SHA:** `3a06655ec5879deb17d506ccf8d62ebb37865fc9`
- **Author:** gemini-cli-robot
- **Title:** Backlog Health & Stale Policy Optimization
- **Diff size:** +123 / -25 across 3 files
- **Drip:** drip-292

## Files changed

- `.github/workflows/gemini-scheduled-stale-issue-closer.yml` (+58/-25) — adds backlog-age awareness and tightens the existing closer logic.
- `.github/workflows/stale.yml` (+1/-0) — single-line policy tweak to remove `help wanted` "staleness immunity".
- `tools/gemini-cli-bot/metrics/scripts/backlog_age.ts` (+64/-0) — same backlog-age script that also appears in PR #26344.

## Specific observations

- This PR overlaps significantly with #26344 (same author, same backlog_age.ts file, same repo health theme). They will conflict at merge — whichever lands second will need a rebase. Recommend the two PRs be sequenced explicitly (one rebased on top of the other) or squashed into a single landing.
- `stale.yml:+1/-0` removing `help wanted` from staleness immunity is the most behaviourally significant single-line change in this PR. The PR body justifies it ("largely due to staleness immunity for help wanted items"), and the data point ("2342 open issues") supports it. Worth getting explicit maintainer sign-off — community contributors who tagged issues `help wanted` in good faith may see them auto-close, which is socially expensive even if technically correct.
- `gemini-scheduled-stale-issue-closer.yml:+58/-25` — smaller delta than #26344's (+102/-76) on the same file, suggesting this PR is a *subset* of the more comprehensive change in #26344. If both were authored by the same bot run, prefer landing #26344 and closing this one as superseded — otherwise reviewers will end up reconciling two diffs against the same workflow file.
- `backlog_age.ts` (+64/-0) — identical purpose to the +65 version in #26344. Confirm via `git diff` that the *contents* are identical too; if they diverge by even one line, the second-merged PR's tests will surprise.
- The PR body's framing ("Survivorship Bias", concrete counts of 2342 issues / 442 PRs) is excellent — it justifies the change in terms a reviewer can verify, not just bot-generated boilerplate.

## Verdict: `needs-discussion`

Mechanically the changes look fine, but this PR materially overlaps with PR #26344 from the same bot author. Landing both as-is will cause a merge conflict on `gemini-scheduled-stale-issue-closer.yml` and a duplicate `backlog_age.ts`. Maintainers should pick one (likely #26344, which is the superset) and close the other as superseded — or explicitly rebase one on top of the other. The `help wanted` immunity removal in `stale.yml` also deserves an explicit "yes, we're doing this" from a human maintainer before either PR lands.
