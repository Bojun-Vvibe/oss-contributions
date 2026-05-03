# block/goose PR #8959 — chore(deps): bump dawidd6/action-download-artifact from 12 to 21

- Link: https://github.com/block/goose/pull/8959
- SHA: `49cbeac7e391981c9fea086d6b4a5664241c94f1`
- Author: app/dependabot
- Stats: +1 / −1, 1 file

## Summary

Dependabot bump of the `dawidd6/action-download-artifact` GitHub Action from v12 to v21 in the `update-health-dashboard.yml` workflow. The PR pins by commit SHA (`b6e2e70617bc3265edd6dab6c906732b2f1ae151`) with a `# v21` comment, which matches Dependabot's standard policy and the repo's existing convention for the same action at v12.

## Specific references

- `.github/workflows/update-health-dashboard.yml` L29–L30: the only change is the action ref. Previous pin was `0bd50d53a6d7fb5cb921e607957e9cc12b4ce392 # v12`, new pin is `b6e2e70617bc3265edd6dab6c906732b2f1ae151 # v21`. SHA pinning is preserved (this is the secure way to consume third-party Actions, since tags are mutable). Good.
- The downstream `with:` block (`name: health-metrics`, `path: .`) is unchanged. Worth a quick check against the v12→v21 changelog to confirm no input was renamed/removed; the v21 release notes published by `dawidd6/action-download-artifact` should be reviewed for breaking input changes (e.g. `workflow_conclusion` defaults, `if_no_artifact_found` behaviour, `name` matching semantics).
- Single-action bump in a single workflow with no other call sites of this action (per the diff scope) — blast radius is limited to the health-dashboard workflow. If that workflow runs on `schedule:`, a stealth break wouldn't be caught by any PR's required checks; manually trigger the workflow once after merge to confirm.

## Verdict

verdict: merge-after-nits

## Reasoning

Mechanically the change is correct (SHA-pinned, comment carries the version, single-line diff). The 9-major-version jump (v12 → v21) deserves one manual confirmation: skim the v13–v21 release notes for input-shape changes, and trigger the workflow manually post-merge to confirm the dashboard still renders. If both are clean, this is a routine merge.
