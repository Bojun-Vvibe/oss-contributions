# google-gemini/gemini-cli #26274 — fix(cli): allow installing extensions from ssh repo

- SHA: `235e3d9351e2d889f93fd51be48c6ecdba4ac32e`
- State: OPEN, +2/-1 in 1 file
- Files: `packages/cli/src/config/extension-manager.ts`
- Fixes: #26273

## Summary

One-line addition to `inferInstallMetadata()` so `ssh://` URLs are recognized as git sources alongside `git@`, `sso://`, `github:`, and `gitlab:`. Some self-hosted git servers expose only the `ssh://host/path` form, which currently fails the install-source classifier.

## Notes

- `packages/cli/src/config/extension-manager.ts:1300-1304` — adds `source.startsWith('ssh://')` to the existing OR chain. Correct and minimal.
- The `ssh://` scheme is the canonical git transport per `git help clone` (see "GIT URLS"); it's odd it wasn't already on the list. The fix is unambiguous.
- No tests in this PR. The PR's own checklist marks Added/updated tests as not done. The surrounding code path almost certainly has a tabular test for `inferInstallMetadata` source classification — adding one row for `ssh://example.com/repo.git` would be ~5 lines and prevent regression.
- Validation checklist in the PR body is mostly unchecked; only `npm run` on Windows/Linux is checked. For a one-line classifier change this is acceptable, but a unit test would replace the manual matrix entirely.
- No security concern: this only affects which sources are *attempted* as git installs; the actual `git clone` machinery downstream already handles the scheme correctly (presumably — would be worth a quick sanity check that the install pipeline doesn't have an allowlist further down that re-rejects `ssh://`).

## Verdict

`merge-after-nits` — add one unit test row for `ssh://` to whatever table drives `inferInstallMetadata` classification, then merge. The fix itself is correct.
