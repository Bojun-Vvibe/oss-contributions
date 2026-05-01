# QwenLM/qwen-code#3776 — feat(installer): add standalone archive installation [skip ci]

- **PR**: https://github.com/QwenLM/qwen-code/pull/3776
- **Head SHA**: `eb2a9a8beff27991b3718f9d879e6471977eb7de`
- **Size**: +2312 / -809, 14 files (release workflow, install scripts for both `.sh` and `.bat`, design docs, packaging script, test plan)
- **Verdict**: **merge-after-nits**

## Context

The current one-line installer installs through npm and requires a working Node.js + npm environment. This PR adds code-server-style standalone archive distribution: per-platform `qwen-code-{darwin,linux,win}-{arm64,x64}.{tar.gz,zip}` artifacts that bundle `dist/cli.js`, required runtime assets, and a private Node.js runtime (no system Node required). Installer defaults to `--method detect` (try standalone → fall back to npm), with explicit `--method standalone` and `--method npm` overrides plus a `--archive /path/to/archive` offline-install path.

## What's right

- **Release workflow at `.github/workflows/release.yml:382-426`** — new "Build Standalone Archives" step that, for each of `{darwin-arm64, darwin-x64, linux-arm64, linux-x64, win-x64}`, downloads the matching Node.js binary from `https://nodejs.org/dist/v${NODE_VERSION}/`, **verifies it against `SHASUMS256.txt`**, and feeds it into `npm run package:standalone -- --target ... --node-archive ... --version ${RELEASE_VERSION}`. The `awk -v name="${archive_name}" '$2 == name { print }'` line at `:402` correctly extracts the matching checksum line by exact-name match (not substring), and `printf '%s\n' "${checksum_line}" | (cd "${RUNTIME_DIR}" && sha256sum -c -)` verifies before extraction. Fail-loud `set -euo pipefail` at the top means any download or checksum failure aborts the entire release.
- **Asset names intentionally exclude the version.** Design doc at `.qwen/design/2026-04-30-standalone-installer-design.md:60-63` documents the choice: lets the installer use GitHub's `releases/latest/download/<asset>` URL without an API call. Versioned installation still possible via `releases/download/vX.Y.Z`. Right tradeoff for a default-detect installer.
- **`SHA256SUMS` artifact uploaded alongside the binaries** at `:460`. Mandatory for the `--method standalone` and `--method detect` paths to verify the downloaded archive before extraction (per design doc `:120`: "Remote standalone installs require checksum verification").
- **Aliyun OSS mirror is documented but secondary.** Design doc at `:108-113` pins the discipline: "All mirrors must serve byte-identical artifacts and the same `SHA256SUMS`." The installer doesn't do geo-detection (explicit non-goal at `:30`); user picks the mirror via flag.
- **Safety carve-outs are explicit** in design doc `:118-124`: installer does NOT modify npm prefix / npmrc / shell profiles / persistent PATH / auto-start `qwen`. Just extracts to `~/.local/lib/qwen-code` (Unix) or `%LOCALAPPDATA%\qwen-code` (Windows) and creates a shim at `~/.local/bin/qwen` / `%LOCALAPPDATA%\qwen-code\bin\qwen.cmd`. The "if PATH doesn't include shim dir, *print* the dir to add" pattern is the right call (no surprise mutations).
- **Test plan at `.qwen/e2e-tests/2026-04-30-standalone-installer-test-plan.md`** (87 lines, new) plus `scripts/tests/install-script.test.js` (218 lines, new) — coverage includes the static "installer keeps expected methods, doesn't reintroduce Node/NVM auto-install" property and shell-smoke tests with faked `curl`/`tar`/`npm`/`node`/`qwen` binaries.
- **`gh release create` updated at `:460-461`** to include `dist/standalone/qwen-code-*` and `dist/standalone/SHA256SUMS` alongside `dist/cli.js`. Atomic with the rest of the release.
- **`.gitignore` carve-outs at `:36-39`** allow `.qwen/design/` and `.qwen/e2e-tests/` to be committed (everything else under `.qwen/` is dev-state).

## Risks / nits

- **No code-signing or notarization.** Design doc at `:30` explicitly defers this ("Solve code signing or notarization in the first implementation" is a non-goal). The macOS install path will produce a Gatekeeper warning on first launch — worth surfacing this in the user-facing install instructions so users know to expect it. Linux is fine, Windows SmartScreen may also warn.
- **Native module parity caveat.** Design doc at `:34`: "Guarantee parity for optional native modules such as `node-pty` and clipboard packages" is a non-goal. Worth a doc note in the install README that standalone-installed `qwen` may degrade certain features (specifically: which features?) compared to npm-installed.
- **`set -euo pipefail` plus `local checksum_line; checksum_line="$(awk ...)"` pattern at `:399-401`.** The `local` declaration *separate* from the assignment is correct — the inverse pattern (`local foo="$(false)"`) silently swallows the `false` exit code because `local` itself succeeds. The PR has it right; worth a comment so a future contributor doesn't "simplify" it.
- **`download_node` runs in a serial loop at `:418-422`.** Five Node downloads × ~30 MB each = ~150 MB serial download. A `wait` + background-curl pattern would shave minutes off the release CI time. Cosmetic optimization, not a correctness issue.
- **No test-pin that `npm run package:standalone` produces an archive layout matching the design doc spec.** Design doc `:46-58` specifies `qwen-code/{bin/qwen, bin/qwen.cmd, lib/cli.js, node/..., package.json, README.md, LICENSE, manifest.json}`. The packaging script at `scripts/create-standalone-package.js` (376 lines, new) presumably enforces this but a separate test that *unzips a produced archive and asserts the layout* would prevent silent drift if someone refactors the packaging script.
- **`design/` and `e2e-tests/` markdowns committed under `.qwen/`** — the existing `.gitignore` carve-out for `.qwen/skills/**` and `.qwen/agents/**` was being preserved by allowlist; the new `!.qwen/design/` + `!.qwen/e2e-tests/` follows the same pattern. Worth confirming the directory structure is actually intended to live in-repo permanently (vs being moved to `docs/` once the feature is GA) — design docs in `.qwen/` is a non-standard location for project docs.

## Verdict

**merge-after-nits.** Well-scoped distribution-channel addition with the right safety discipline (no PATH mutation, no auto-start, mandatory checksum verification, npm fallback preserved). Address the code-signing / native-modules / archive-layout-test notes before tagging GA; the parallelization of the Node download loop and the design-docs-location question can be follow-ups.
