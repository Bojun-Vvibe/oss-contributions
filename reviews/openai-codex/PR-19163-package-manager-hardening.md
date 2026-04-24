# PR-19163 — Harden package-manager install policy

[openai/codex#19163](https://github.com/openai/codex/pull/19163)

## Context

Sweeping supply-chain hardening across the repo's three install
boundaries:

1. **`.devcontainer/Dockerfile.secure` + new `.devcontainer/codex-
   install/`** — replaces `npm install -g
   "@openai/codex@${CODEX_NPM_VERSION}"` (where `CODEX_NPM_VERSION`
   defaulted to `latest`) with a pinned `package.json` (`@openai/
   codex: 0.121.0`), a committed `pnpm-lock.yaml`, and a strict
   `pnpm-workspace.yaml` (`minimumReleaseAge: 10080`,
   `blockExoticSubdeps: true`, `strictDepBuilds: true`,
   `trustPolicy: no-downgrade`, `allowBuilds: {}`). Install becomes
   `corepack pnpm install --prod --frozen-lockfile`.
2. **`codex-cli/Dockerfile` + new `codex-cli/container-install/`** —
   same pattern applied to the published CLI Docker image. Replaces
   `npm install -g codex.tgz` with a frozen-lockfile pnpm install
   into `/opt/codex-install` plus a symlink at `/usr/local/bin/codex`.
3. **`codex-cli/scripts/stage_container_package.sh` (+117)** — new
   helper that builds a self-contained `dist/codex.tgz` from either
   a local musl binary or the latest successful `rust-release` Linux
   artifact. `build_container.sh` is updated to call it instead of
   the broken `pnpm run build` (which never existed in
   `codex-cli/package.json`).
4. **`sdk/python/uv.lock` (+711) + `sdk/python-runtime/uv.lock`** —
   new committed lockfiles + uv age/index settings in the two
   Python SDK packages; getting-started docs swap `pip install` for
   `uv sync`.
5. `bin/codex.js` — drops the embedded `npm install -g @openai/codex
   @latest` reinstall hint in favor of `reinstallCodexHint()` that
   only names the user's package manager.

## Strengths

- **Pinning + lockfile + age gate stack is the right shape** for a
  CLI distributed via npm. `minimumReleaseAge: 10080` (7 days)
  blocks freshly-published malicious versions of transitive deps
  from landing in a container build seven days before they hit
  npm-malware feeds.
- **`CODEX_NPM_VERSION=latest` → `0.121.0`** is the headline fix.
  Floating `latest` tags in a Dockerfile mean the same `Dockerfile`
  produces a different image every day, with no audit trail. Pinning
  it to a digest-equivalent version + lockfile makes the build
  reproducible and the SBOM coherent.
- **Dual install boundary** (`/opt/codex-install` outside
  `/usr/local/share/npm-global`) keeps the install root immutable
  and read-only by the unprivileged user — a runtime privilege
  escalation can't trivially overwrite the installed CLI.
- **`reinstallCodexHint()` no longer suggests `npm install -g
  @openai/codex@latest`** in error messages. Users following the
  hint were silently bypassing the lockfile they were supposed to be
  using.
- **`stage_container_package.sh` falls back to a CI artifact** when
  no local musl build exists — the comment about `build_container.
  sh` already being broken on `main` (calling a non-existent `pnpm
  run build`) is a separate latent bug that this PR also fixes.

## Concerns / risks

- **`test "$(node -p "require('/opt/codex-install/package.json').
  dependencies['@openai/codex']")" = "${CODEX_NPM_VERSION}"`** —
  this guard is run *inside* the Docker layer to assert the
  `package.json` and the `ARG` agree, but `corepack pnpm install`
  hasn't run yet at that point. If the `ARG` is overridden at build
  time without also editing the committed `package.json`, the build
  fails — good. But the failure message will be a bare exit code,
  not a useful diff. Worth wrapping with an explicit `echo "ARG
  CODEX_NPM_VERSION=...does not match package.json='${have}'"` on
  failure.
- **`trustPolicyIgnoreAfter: 10080`** in `pnpm-workspace.yaml`
  expires the trust policy after 7 days, which on a long-lived
  Docker base image cache means rebuilds older than a week will
  start trusting newly-published packages. Document the
  expectation that the install image is rebuilt at least weekly,
  or set this longer.
- **`/opt/codex-install/node_modules` rm-cleanup is selective** —
  the diff strips `.cache`, `tests/`, `docs/`. Worth also pruning
  `*.md`, `LICENSE.txt` duplicates, and any `.git*` directories
  that pnpm sometimes leaves; small but it's an SBOM noise source.
- **`.gitignore` adds `.venv/`** which is unrelated to the package-
  manager hardening theme; flag for the reviewer that this lands
  in the same commit. Not blocking but cluttering.
- **The `bin/codex.js` hint regression**: previously the message
  was actionable (`npm install -g @openai/codex@latest`), now it's
  prose ("Reinstall Codex with npm.") that doesn't tell the user
  the package name. For CI failures this is worse-DX. Either
  restore `@openai/codex` in the message or link to docs.
- **No CI assertion that the lockfiles stay in sync** with
  `package.json`. The `pnpm install --frozen-lockfile` run *inside*
  the Docker build will catch drift, but that's a 5–10 minute
  signal vs a 10-second `pnpm install --frozen-lockfile` job in CI.
  Add a fast lockfile-integrity job to the PR template's required
  checks.

## Verdict

**Approve.** The supply-chain posture improvement is real and the
two Dockerfiles + the python uv lockfiles all move in the same
direction. Follow-ups are observability/UX-grade, not blocking:
better failure messages from the version-equality guard, restore
the package name in the reinstall hint, and add a fast `pnpm
install --frozen-lockfile` lockfile-drift CI job so the slow
container build isn't the only canary.
