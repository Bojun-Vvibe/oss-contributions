# QwenLM/qwen-code #3705 — fix(ci): ignore manual version for preview releases

- PR: https://github.com/QwenLM/qwen-code/pull/3705
- Author: doudouOUC (jinye)
- Head SHA: `1cdc92f2542f`
- Base: `main` · State: OPEN · Fixes #3686
- Diff: 0+/6- across 2 files (`.github/workflows/release-sdk.yml`, `.github/workflows/release.yml`)

## Verdict: merge-as-is

## Rationale

- **Right diagnosis of a real "stable-version input crashed the preview path" bug.** The PR body cites failing run `25032972161` where `--preview_version_override=v0.1.7` was rejected by the version script with `Invalid preview_version_override: 0.1.7. Must be in X.Y.Z-preview.N format`. The structural cause is the conditional at `release-sdk.yml:117` and `release.yml:117`: when `IS_PREVIEW=true` *and* `MANUAL_VERSION` is non-empty, the workflow forwarded the manual version into `--preview_version_override`. But the workflow's own documentation (per the PR body: "Preview releases are documented as auto-generated") says preview versions should be auto-derived. The conditional was an unintended carry-over from the stable path that handled `MANUAL_VERSION` correctly.
- **Symmetric three-line deletion across both workflows is the right shape.** Both `release-sdk.yml:117-119` and `release.yml:117-119` had identical structure: `if [[ -n "${MANUAL_VERSION}" ]]; then VERSION_ARGS+=("--preview_version_override=${MANUAL_VERSION}"); fi` nested inside the `IS_PREVIEW=true` arm. Deleting the same three lines in both files keeps the workflows in lockstep — which is the right invariant since they share the same version-derivation contract. Asymmetric fix (delete in one, keep in the other) would have been a much harder-to-detect drift.
- **The stable-release arm (`release-sdk.yml:120-126`-ish) is untouched.** The same `if [[ -n "${MANUAL_VERSION}" ]]` pattern in the `else` (stable) branch correctly forwards `MANUAL_VERSION` for stable releases, where operators *do* need to override the version explicitly. Surgical deletion preserves the stable path; only the preview path gets the auto-derive treatment it was documented to have.
- **PR body cites the exact failed run, the validation command, and the expected behavior.** `gh run view 25032972161 --repo QwenLM/qwen-code --log-failed` is the canonical evidence anchor. `node packages/sdk-typescript/scripts/get-release-version.js --type=preview` validates the auto-derivation works locally without the override. That's the right amount of evidence for a CI workflow fix.

## Nits / follow-ups

- **The manual-version input is now silently ignored on the preview path.** If an operator types `version: v0.1.7-preview.5` into the workflow_dispatch UI expecting it to be honored, the workflow now drops it without warning. A `echo "::warning::MANUAL_VERSION ignored on preview path; preview versions are auto-derived"` line right above the deleted block (so it fires only when both `IS_PREVIEW=true` *and* `MANUAL_VERSION` is non-empty) would close the silent-drop hole. Two-line addition.
- **The workflow_dispatch input description for `version` should be updated** to read something like `Manual version override (ignored when type=preview; preview versions are auto-derived)`. Without that, operators will keep typing into the field and being surprised when it's dropped. Doc-only follow-up.
- **No test coverage for workflow logic** — but that's a structural CI-yaml reality, not a PR-specific gap. The validation strategy (re-run via workflow_dispatch in a fork, confirm preview path no longer crashes) is the only practical anchor.
- **`MANUAL_VERSION` could legitimately be `X.Y.Z-preview.N` format and *should* be honored** for the rare case of pinning a specific preview suffix. The PR's framing — "manual `version` input is now ignored for preview releases" — is the simpler/safer choice, but a future iteration could add a regex check (`^[0-9]+\.[0-9]+\.[0-9]+-preview\.[0-9]+$`) and only forward when the format matches. Out of scope here; current fix is correctly conservative.

## What I learned

The "stable-path conditional was copy-pasted into the preview path arm" bug is a recurring shape in CI workflow yaml — it's structurally symmetric across the two arms in a way that *looks* right (both arms handle `MANUAL_VERSION`), but the semantics of the two paths are different. Stable releases need operator-pinned versions for hotfix-and-cherry-pick discipline; preview releases need auto-derived versions for chronological ordering and uniqueness. The fix shape — delete the symmetric block from the preview arm, leave the stable arm intact — preserves the *right* asymmetry. The lurking-symmetry test for "is this conditional correct in both arms?" is "what's the failure mode if a user types into the field?" — for stable, it's "their override wins" (correct); for preview, it was "their override crashes the run" (wrong). When the failure modes are asymmetric, the conditional should be too.
