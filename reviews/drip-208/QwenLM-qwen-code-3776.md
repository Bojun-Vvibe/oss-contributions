# QwenLM/qwen-code#3776 — feat(installer): add standalone archive installation

- PR: https://github.com/QwenLM/qwen-code/pull/3776
- Head SHA: `eb2a9a8bef...`
- Author: yiliang114
- Files: 14 changed, ~3600 LOC (release workflow, two new design docs, packaging script, install scripts for unix+windows, tests)

## Context

Today's qwen install path is npm-only (`npm install -g @qwen-code/qwen-code`), which forces users to provision a Node toolchain first. This PR adds a parallel "standalone" install method that ships a self-contained archive bundling a pinned Node runtime + the `qwen` CLI. The unix/Windows installers gain a `--method` flag (`detect`, `npm`, `standalone`); `detect` prefers the standalone archive when one matches the host target, falls back to npm otherwise. Release workflow gains a build-and-attach step that produces four archives: `qwen-code-{darwin-arm64,darwin-x64,linux-arm64,linux-x64}.tar.gz` plus a Windows zip.

## Design

Four moving parts:

1. **`scripts/create-standalone-package.js`** (new, ~350 LOC) — accepts `--target`, `--node-archive`, downloads (or accepts a pre-staged) Node runtime archive, extracts it, copies the built `qwen` distribution next to it, computes a sha256 (`createHash('sha256')` at line 949), emits `qwen-code-<target>.tar.gz` for unix and `.zip` for Windows. Wired into `package.json:591` as `"package:standalone": "node scripts/create-standalone-package.js"`.
2. **Release workflow `.github/workflows/release.yml`** — adds a job that loops `download_node` over the four target/arch pairs and runs the packager. The download step verifies via `printf '%s\n' "${checksum_line}" | (cd "${RUNTIME_DIR}" && sha256sum -c -)` against Node's published `SHA256SUMS`, so a mirror swap or tampered archive fails the build.
3. **Unix installer `scripts/installation/install-qwen-with-source.sh`** — gains `verify_checksum()` (handler.py:2887-2945) which:
   - Resolves the SHA256SUMS source (release asset or mirror),
   - Picks `sha256sum` or `shasum -a 256` based on `command_exists`,
   - Computes actual via `sha256_file`,
   - Compares case-insensitively, errors loudly with `log_error "SHA256SUMS not found; cannot verify remote archive."` if the SUMS file is missing.

   This is gated *before* extraction, so a corrupted/tampered archive never gets to `tar -xzf`.
4. **Windows installer `scripts/installation/install-qwen-with-source.bat`** — parallel changes (the diff at line 1906 removes the legacy `node-v!NODE_VERSION!.msi` direct-download path and replaces it with the standalone-archive flow).

Tests at `scripts/tests/install-script.test.js` are static-string assertions (`expect(script).toContain('verify_checksum()')`, `expect(script).toContain('SHA256SUMS not found; cannot verify remote archive')`, `expect(script).not.toContain('node-v!NODE_VERSION!')`) — these protect against a future refactor that quietly removes the integrity-verification path. Worth their cost.

## What's good

- **Mandatory checksum verification of the Node runtime, not optional** — the installer aborts when the SHA256SUMS file is missing. This is the right default for a tool that downloads and executes a Node binary.
- **No automatic `qwen` startup after install** — the design doc at `.qwen/design/2026-04-30-standalone-installer-design.md:118` explicitly says "Never start `qwen` automatically from the installer." Correct security/UX posture.
- **No code signing / notarization in v1** — explicitly scoped out at `.qwen/design/...:125`. Honest scoping; users will get the macOS Gatekeeper prompt on first run, which is acceptable for v1.
- **Release-asset URL via `releases/latest/download/<asset>`** — avoids burning GitHub API quota on installer runs (mentioned at line 143 of the design doc). Standard pattern.
- **Per-target node_modules left out of v1** — design doc at line 128 acknowledges this; the standalone bundles whatever was built on the build host. If a native dep ever creeps into qwen-code's runtime deps, this will need to be revisited (per-target `npm install --target_arch=...` in the packager). Worth tracking as a known v2 item.

## Risks / nits

- **`installer.sh` modifies user PATH only on the current process, not persistently** — design doc at line 195: "The installer may add the command directory to the current process PATH for the current shell, but the user is expected to update their shell init for persistent access." This is the right safe default (don't munge `.zshrc` without consent), but the post-install message must clearly print the exact `export PATH=...` line for the user to copy. Confirm the diff at `install-qwen-with-source.sh` does this prominently, not buried after a 20-line success banner.
- **Windows installer parity is harder to audit** — the `.bat` file changes drop the MSI flow and replace it with the archive flow, but Windows lacks `sha256sum` natively. The diff should use PowerShell `Get-FileHash -Algorithm SHA256` (or `certutil -hashfile`); confirm one of those is wired in. The static test at line 3486 only asserts the *removed* MSI path, not the *presence* of a Windows SHA256 verifier.
- **Cleanup posture** — design doc at line 222: "The installer only deletes temporary extraction directories and the previous install dir." This needs a careful audit because a misimplemented "previous install dir" detection that returns `$HOME` would be catastrophic. The shell tests should include an assertion that the previous-install resolution refuses to delete anything outside `${RUNTIME_DIR}` / the canonical install root.
- **`stat` flag portability** — if the installer uses `stat -c` (GNU) it will fail on macOS (BSD `stat -f`). Quick audit of any `stat` usage in the new shell script.
- **Mirror flag (`--mirror`)** — the design doc lists `--mirror` as a CLI flag for choosing a Node distribution mirror. Make sure the test asserting "exposes `--method`, `--mirror`" at line 262 of the plan doc validates the mirror URL is HTTPS-only and on an allowlist (or at least logs prominently when a non-default mirror is used). Otherwise this is a supply-chain side door for someone who can MITM the install command.
- **Three new design/plan/test-plan docs under `.qwen/design/` and `.qwen/e2e-tests/`** check in as part of this PR. Useful for review trail; consider whether they should live under `docs/contrib/` long-term so they're discoverable, or be deleted post-merge if they were just scratch.

## Verdict

**request-changes** — the bones (checksum-mandatory, no-auto-start, scoped-out signing) are correct. The blockers before merge:

1. Confirm Windows installer has an actual SHA256 verifier (`Get-FileHash` or `certutil`); add a static test asserting its presence.
2. Audit the "delete previous install dir" path against a `$HOME`-or-shorter pathological case; add a refusal test.
3. The `--mirror` flag needs at minimum an HTTPS-only check + a loud log line so users notice if their install command is pulling Node from a non-canonical mirror.
4. Decide whether `.qwen/design/*.md` and `.qwen/e2e-tests/*.md` belong in `docs/contrib/` post-merge.

Once those are addressed this is solid — the conceptual model (separate method flag, npm stays as a parallel option, mandatory integrity verification, no auto-start) is exactly right.
