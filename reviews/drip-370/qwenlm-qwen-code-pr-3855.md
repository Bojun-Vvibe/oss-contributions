# QwenLM/qwen-code PR #3855 — feat(installer): verify installation release assets

- URL: https://github.com/QwenLM/qwen-code/pull/3855
- Head SHA: `b1757402fdb39f68e3ed6d188d9b57bafa445143`
- Author: stacked PR (4th of installer follow-up series), stacked on #3853
- Size: +4837 / -796 (16 files)
- Related: #3728

## Verdict
`merge-after-nits`

## Rationale

The PR's central change at `.github/workflows/release.yml:382-391` correctly orders the new asset
pipeline: build standalone archives → run the installer-asset packager (which rewrites `SHA256SUMS`
to cover both standalone archives and copied installer scripts) → run
`npm run verify:installation-release -- --dir dist/standalone` *before* `gh release create` uploads
anything. The placement is the load-bearing decision — verifying after upload would still have
caught problems but only after exposing them publicly. The expanded `gh release create` argv at
`release.yml:425-431` consistently lists the new files (`qwen-code-*` standalone archives, the three
installer entrypoints, and `SHA256SUMS`) so the verifier's input set and the publisher's output set
stay in sync. Documentation churn (README + `docs/users/overview.md` + `docs/users/quickstart.md`)
correctly drops the "Run as Administrator" caveat for Windows now that the installer is standalone-first
and adds the offline `--archive PATH` flag to the user-facing install notes — these are real
behavioral changes, not cosmetic.

The new `scripts/verify-installation-release.js` is the right shape: it walks the release dir,
validates `SHA256SUMS` entries against on-disk bytes, asserts byte-equality between `install` and
`install-qwen.sh` (the two are intentionally identical so naming-by-platform is a UX-only split,
and a drift bug there would silently ship divergent installers under two URLs), and for remote-mode
HEAD-checks the large standalone archives while full-checksum-checking the small installer scripts —
the asymmetry is correct because re-downloading hundreds of MB of standalone archives in CI just to
re-hash them adds minutes and catches nothing the local `SHA256SUMS` step didn't already catch. The
lockfile step `check:lockfile` is preserved and the `package.json` scripts addition (`package:standalone`,
`package:standalone:release`, `package:installation-assets`, `verify:installation-release`) gives
the workflow stable named entrypoints rather than ad-hoc `node scripts/...` lines.

Nits: (1) `.github/workflows/release.yml:382-388` — the three new steps lack `if: success()` /
explicit `continue-on-error: false`; default behavior is correct, but being explicit makes the
"verify before publish" intent unambiguous to anyone editing the workflow later. (2) The verifier
only HEAD-checks remote archives — for the canary release at minimum, a one-shot full-archive
checksum mode (`--remote-full-checksum`) would close the residual gap where a CDN serves a wrong-but-
existing object. (3) The matrix table in the PR body shows Windows / Linux as ⚠️ (CI-only) for
`npm run` and `npx` — given this PR ships *Windows-specific* installer logic (`install-qwen.bat`,
`scripts/installation/install-qwen-with-source.bat`), a Windows runner row in the workflow that
exercises the verifier against a downloaded canary asset would prevent a class of "looks fine on
mac" regressions in the rotation. None of these block; the verifier-before-publish ordering and the
sym-equality guard are the parts that actually matter and they're correct.
