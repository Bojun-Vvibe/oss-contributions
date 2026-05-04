# google-gemini/gemini-cli#26462 — ci(release): build and attach unsigned macOS binaries to releases

- **URL**: https://github.com/google-gemini/gemini-cli/pull/26462
- **Head SHA**: `2ffa21741aab`
- **Diffstat**: +185 / -4
- **Verdict**: `merge-after-nits`

## Summary

Adds a `build-mac` job to the nightly, manual, and promote release workflows that produces unsigned macOS `x64` and `arm64` binaries (`gemini-darwin-arm64-unsigned`, `gemini-darwin-x64-unsigned`) and attaches them as release assets. Closes the per-issue request for unsigned macOS distribution.

## Findings

- `.github/workflows/release-manual.yml:+51 / -0`, `release-nightly.yml:+51 / -0`, `release-promote.yml:+66 / -2` — three near-identical `build-mac` job definitions. The duplication is the biggest reviewer concern: any future change (Node version, signing later, runner image bump) needs to be applied in three places. Strongly recommend extracting into `.github/actions/build-unsigned-mac/action.yml` (the repo already uses composite actions per `.github/actions/publish-release/action.yml`). Not blocking for this PR, but worth filing as a follow-up.
- `.github/workflows/build-unsigned-mac-binaries.yml:+1 / -1` — single-line touch on a pre-existing reusable workflow. Unclear from the diff snippet whether the new `build-mac` jobs *call* this reusable workflow or duplicate its logic. If they duplicate it, the reusable workflow is the natural extraction target above. Reviewer should clarify.
- `.github/actions/publish-release/action.yml:+16 / -1` — appends the two macOS unsigned artifacts to `RELEASE_ASSETS` when creating the GitHub release. Reasonable; just verify the artifact-name strings (`gemini-darwin-arm64-unsigned`, `gemini-darwin-x64-unsigned`) are pinned as constants in one place rather than repeated literals across the three workflows + this action.
- **Security/UX consideration:** publishing *unsigned* macOS binaries as release assets is fine, but they will trigger Gatekeeper quarantine warnings on download. The release notes template (or the asset filenames themselves — they already include `-unsigned`, good) should make clear users will need `xattr -d com.apple.quarantine ./gemini` or similar. Worth a one-line README / docs update; not in scope of this PR's diff but worth flagging.
- PR validation checklist is mostly unchecked — author notes only that they propose to dry-run the manual workflow on the branch. Reviewer should require evidence of a successful dry-run before merging, since CI changes can't be tested in PR alone.

## Recommendation

Plumbing is correct and the artifact naming is honest about the unsigned status. Extract the duplicated `build-mac` job into a composite action (or call the existing reusable workflow), require a manual-release dry-run as merge gate, and add the Gatekeeper-quarantine note to the user-facing docs in a follow-up PR.
