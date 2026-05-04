# QwenLM/qwen-code #3828 — feat(installer): publish release installer assets

- **Head SHA:** `ec654dd87c818382770a785b579318baceadf1d8`
- **Size:** +245 / -21 across 11 files
- **Verdict:** **merge-after-nits**

## Summary
Publishes `install-qwen.sh` and `install-qwen.bat` as GitHub Release assets,
includes them in `SHA256SUMS`, and updates the release workflow plus user-facing
install docs to point at the release-asset URLs instead of the OSS bucket. Also
adds `--version vX.Y.Z` pinning support so users can install a specific
standalone release.

## Strengths
- The release workflow change (`.github/workflows/release.yml` around line 384)
  builds the installer assets right after the standalone build and before the
  `gh release create` invocation, so they ride along in the same release-create
  call. Simple, atomic.
- Updating `README.md` and `docs/users/overview.md` to use
  `https://github.com/QwenLM/qwen-code/releases/latest/download/install-qwen.sh`
  removes the dependency on the third-party OSS bucket, which is good for both
  reliability and supply-chain transparency (releases are signed/attested by GH).
- `--version` pinning is a real ergonomic win for reproducible installs in CI and
  air-gapped relays.

## Nits
- The install one-liner switched from `bash -c "$(curl ...)"` to
  `curl ... | bash`. The two are functionally similar but the piped form makes it
  slightly harder for users to inspect the script before running. Consider
  documenting the inspect-first alternative
  (`curl -fsSL .../install-qwen.sh | less`) right next to the one-liner.
- Adding installer scripts to `SHA256SUMS` is the right call; please verify the
  installer scripts themselves are deterministic across runs (no embedded
  timestamps, no `git describe` interpolation that changes with build context),
  otherwise the checksum claim in `SHA256SUMS` becomes misleading.
- `package:installation-assets` is referenced in the workflow but the diff does
  not show its `package.json` script definition in the visible slice — confirm
  it exists in the full diff.

## Recommendation
Land after confirming the npm script is wired, installer-script determinism, and a
brief docs note on inspecting the installer before piping to `bash`.
