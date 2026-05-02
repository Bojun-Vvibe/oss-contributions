# QwenLM/qwen-code PR #3776 — feat(installer): add standalone archive installation

- URL: https://github.com/QwenLM/qwen-code/pull/3776
- Head SHA: `eb2a9a8beff27991b3718f9d879e6471977eb7de`
- Related: #3728, #3738
- Verdict: **needs-discussion**

## What it does

Adds code-server-style standalone release archives that bundle a private
Node.js runtime with the Qwen CLI, plus updates the Unix and Windows
installer scripts to (a) prefer standalone archives when available, (b)
fall back to npm if not, and (c) accept a local `--archive` for offline
installs. Also adds three new design/test-plan docs under `.qwen/`.

## Specifics I checked

- **`.github/workflows/release.yml:379-465`** — adds a `Build Standalone
  Archives` job that downloads the matching upstream Node.js distribution
  for 5 targets (`darwin-arm64`, `darwin-x64`, `linux-arm64`,
  `linux-x64`, `win-x64`), verifies the SHA256 against
  nodejs.org's `SHASUMS256.txt`, and packages each via
  `npm run package:standalone`. The downloaded archive is also uploaded
  via `gh release create dist/standalone/qwen-code-* dist/standalone/SHA256SUMS`.
  Asset names omit the version (`qwen-code-darwin-arm64.tar.gz`) so the
  installer can use `releases/latest/download/<asset>`.
- **`.qwen/design/2026-04-30-standalone-installer-design.md`** — clear,
  honest problem statement and explicit non-goals (no single-binary, no
  geo-mirror, no automatic Node install, no code signing in v1, optional
  native modules like `node-pty` may degrade). Install layout uses
  `$HOME/.local/lib/qwen-code` + `$HOME/.local/bin/qwen` on Unix and
  `%LOCALAPPDATA%\qwen-code\` on Windows. The installer explicitly does
  NOT modify shell profiles or auto-start `qwen` — that's the right call.
- Smoke test in PR body uses a fake `node` shim that prints
  `0.0.0-smoke` to validate end-to-end packaging + install without a
  real Node runtime — neat trick, gives confidence the shim wiring is
  correct without depending on network downloads.

## What I like

- Honest scoping. The PR doesn't claim Windows VM or real GitHub Release
  CI smoke; it lists those as "not validated in this pass". That's the
  kind of self-disclosure that makes review tractable.
- SHA256 verification against upstream nodejs.org is mandatory — the
  job exits if the line is missing. No silent "checksum unavailable"
  fallback.
- Mirror flexibility: the design supports both GitHub Releases and an
  Aliyun OSS mirror with the same artifact names and checksums.
- `--method npm` is preserved as an explicit escape hatch, plus a
  fallback when no standalone asset exists for a target.

## Concerns

1. **Optional native modules (`node-pty`, clipboard).** Design doc
   explicitly says these are not parity-tested. `node-pty` powers the
   shell tool / interactive runs in many CLI agents; if the standalone
   archive ships without a working `node-pty` for the target, users may
   silently lose terminal capabilities. Two questions for the author:
   (a) what's the user-visible signal when this happens? (b) is there a
   smoke test that catches the regression at release time, or is it
   "report a bug"?
2. **Windows path for `tar.xz`.** The job uses `tar.gz` for darwin and
   `tar.xz` for Linux. Confirm `npm run package:standalone` on the
   GitHub-hosted Linux runner has `xz-utils` available (Ubuntu runners
   do, but worth pinning).
3. **Asset size.** Bundling Node + Qwen CLI per platform = ~5 archives
   × ~50–80MB each. The PR doesn't quote the final asset sizes or
   whether GitHub Release storage / nightly bandwidth is a concern.
4. **`.qwen/design/` and `.qwen/e2e-tests/` whitelisted in
   `.gitignore`.** New `!.qwen/design/` and `!.qwen/e2e-tests/`
   negations. Fine as a convention, but worth deciding now whether
   *every* design doc lives in the repo (vs. an external wiki). Once
   you start, removing later is awkward.
5. **The CLI smoke uses `chmod +x` on a shell script and treats it as
   `node`.** Clever, but if a maintainer reruns the smoke without LC_ALL
   set or with BSD `tar`, behaviors may diverge. Worth pinning to a
   specific OS in CI to reproduce.
6. **No GPG / cosign signing.** Design doc lists this as a non-goal for
   v1, but for a one-line installer running over the network as
   `curl ... | sh`, signing the asset (or at minimum publishing the
   `SHA256SUMS` over a separate channel) is a meaningful security
   improvement to plan for v1.1.

## Verdict rationale

The scope is significant (release pipeline + installer + design docs +
~141 new lines of design + e2e plan). The mechanical changes look
correct and the testing matrix is honest, but item (1) — optional
native module degradation — is a real UX cliff that deserves a
maintainer conversation before this changes the default install path.
Verdict `needs-discussion` on those scope/UX questions; the code is
otherwise close to mergeable as `merge-after-nits`.
