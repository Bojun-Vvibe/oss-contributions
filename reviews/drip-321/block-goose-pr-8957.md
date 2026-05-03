# block/goose PR #8957 — chore(deps): bump azure/login from 2.3.0 to 3.0.0

- Link: https://github.com/block/goose/pull/8957
- SHA: `b160b3f0cacfc0f5fbc0a03d17ee6cb72c9de629`
- Author: app/dependabot
- Stats: +2 / −2, 2 files

## Summary

Dependabot bump of the `azure/login` GitHub Action from v2.3.0 to v3.0.0 across two workflow files. SHA-pinned to `532459ea530d8321f2fb9bb10d1e0bcf23869a43`. Per the v3.0.0 release notes (linked in the PR body), the action was upgraded from Node 20 to Node 24 and dependencies refreshed.

## Specific references

- `.github/workflows/bundle-desktop-windows.yml` (1 occurrence): `azure/login@a457da9ea143d694b1b9c7c869ebb04ebe844ef5  # v2.3.0` → `azure/login@532459ea530d8321f2fb9bb10d1e0bcf23869a43  # v3.0.0`. Note the leading double space before the comment is preserved — cosmetic only.
- `.github/workflows/bundle-goose2.yml` (1 occurrence): `azure/login@a457da9ea143d694b1b9c7c869ebb04ebe844ef5 # v2` → `azure/login@532459ea530d8321f2fb9bb10d1e0bcf23869a43 # v3.0.0`. Comment correctly refreshed from the floating `# v2` to the pinned `# v3.0.0`.
- v2 → v3 is a major-version bump. The runtime upgrade to Node 24 is the headline change. Things to verify before merging:
  1. The runner images used by both workflows include Node 24. `windows-latest` and `ubuntu-latest` both ship Node 24 in current images, so this should be fine.
  2. Any custom `inputs` consumed by the workflow steps that follow login (federated identity / OIDC token refresh, subscription scoping) behave identically in v3 — the v3 release notes do not call out input renames, but a one-off CI run will confirm.
  3. The Azure subscription / tenant secrets the workflows reference are unchanged; this PR only touches the action version.
- Both updates are SHA-pinned. Supply-chain hygiene preserved.

## Verdict

verdict: merge-as-is

## Reasoning

Two-file, four-line action version bump. SHA pins are consistent, the trailing comments correctly reflect v3.0.0 in both files, and the only behavioural change advertised by upstream is the Node 20 → 24 runtime — both `windows-latest` and `ubuntu-latest` already ship Node 24. Low-risk, mergeable as is. If a follow-up PR finds an input regression in the bundle pipelines, that is a separate fix.
