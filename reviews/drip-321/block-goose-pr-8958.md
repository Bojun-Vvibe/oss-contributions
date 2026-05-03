# block/goose PR #8958 — chore(deps): bump pnpm/action-setup from 4.0.0 to 6.0.4

- Link: https://github.com/block/goose/pull/8958
- SHA: `ed0f27688f403d65b78692eba04fe7e344170b9c`
- Author: app/dependabot
- Stats: +5 / −5, 2 files

## Summary

Dependabot bump of the `pnpm/action-setup` GitHub Action from v4.0.0 / v4.4.0 to v6.0.4 across two workflow files. SHA-pinned references are updated to `26f6d4f2c533a43e6b5da0b4a5dd983f98f7b49a`.

## Specific references

- `.github/workflows/goose2-ci.yml` (3 occurrences): three `pnpm/action-setup@fc06bc1257f339d1d5d8b3a19a8cae5388b55320 # v4.4.0` lines update to `26f6d4f2c533a43e6b5da0b4a5dd983f98f7b49a # v6.0.4`. Same pinned SHA across all three — correct.
- `.github/workflows/publish-npm.yml` (2 occurrences): two `pnpm/action-setup@fe02b34f77f8bc703788d5817da081398fad5dd2 # v4` references update to `26f6d4f2c533a43e6b5da0b4a5dd983f98f7b49a # v4`. **Concern:** the SHA is updated to v6.0.4's commit but the trailing comment still reads `# v4`. Either the comment is wrong (most likely — Dependabot normally rewrites it) or this is a dependabot-version-comment glitch. Reviewers should ask Dependabot to refresh and confirm the comment matches the SHA, otherwise future SHA pinning audits will be misled.
- v4 → v6 is a major-version bump; the v5 and v6 release notes (linked in the PR body) add Node 24 support and adjust the `run_install` / package-manager detection behaviour. Confirm the workflow steps that follow `pnpm/action-setup` (e.g. `pnpm install --frozen-lockfile`) still work with the new defaults — particularly any workflow that relies on the older auto-resolution of `packageManager` from `package.json`.
- All five updates are pinned by commit SHA — supply-chain hygiene preserved.

## Verdict

verdict: merge-after-nits

## Reasoning

Pure CI dependency bump with consistent SHA pinning, but the `# v4` trailing comments in `publish-npm.yml` were not refreshed to `# v6.0.4`. That is a small but real correctness issue for future audits and for anyone grep-ing the workflow for action versions. Ask Dependabot to refresh the PR (often re-runs and fixes the comment), or fix the two comments by hand. Once the comments match the SHAs, this is a clean merge — assuming the consuming workflow steps don't rely on v4-era auto-detection behaviour that v6 changed.
