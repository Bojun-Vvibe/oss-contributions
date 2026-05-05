# BerriAI/litellm #27137 — Build Admin UI from source in every Docker image and on PyPI publish

- **Head SHA:** `1e1c55ce5bd330952606df322012af1ba62f18d1`
- **Base:** `litellm_internal_staging`
- **Author:** mateo-berri (Mateo Wang)
- **Size:** +89 / −98 across 6 files
  (`.github/workflows/publish_to_pypi.yml`, `docker/Dockerfile.alpine`,
  `docker/Dockerfile.non_root`, `docker/build_admin_ui.sh`,
  `ui/litellm-dashboard/build_release_ui.sh`,
  `ui/litellm-dashboard/build_ui.sh`)
- **Verdict:** `merge-after-nits`

## Summary

Prep work to remove the checked-in `litellm/proxy/_experimental/out/`
Next.js bundle. Rewrites `docker/build_admin_ui.sh` so it always builds
the dashboard (instead of early-exiting unless an enterprise palette is
present), wires the two Dockerfiles that didn't previously call it
(`Dockerfile.alpine`, `Dockerfile.non_root`), and adds a Node setup +
build step to the PyPI publish workflow. Functionally a no-op today
(the bundle is still committed), but every artifact-producing path now
produces the bundle from source so the follow-up `git rm -rf` is safe.

## What's right

- **The early-exit removal is the actual bug fix.** Old
  `build_admin_ui.sh:11-15` short-circuited unless
  `enterprise/enterprise_ui/enterprise_colors.json` existed, which
  meant OSS Docker builds shipped the *committed* bundle and nothing
  else. New `build_admin_ui.sh:18-23` makes the enterprise palette
  *optional* (copies it if present, logs a default message if not) and
  then runs `npm ci && npm run build` unconditionally. This is the
  prerequisite for deleting the committed bundle.

- **`set -e` is now on.** Old script had `# set -e` commented out and
  a "try except" comment hand-waving over the lack of error handling;
  new script enables `set -e` at line 11 so an `npm ci` failure
  actually fails the Docker build instead of silently shipping a stale
  bundle.

- **Lockfile-aware install.** `build_admin_ui.sh:30-34` uses `npm ci`
  when `package-lock.json` is present, falling back to `npm install`
  otherwise. Right call — `npm ci` is reproducible and faster, but
  fails hard if the lockfile is out of sync, which is what you want
  in CI.

- **HTML restructure logic is centralized.** The `*.html →
  */index.html` rewrite for extensionless `/ui/<route>` routing now
  lives once in `build_admin_ui.sh:42-46`. `Dockerfile.non_root`'s
  bespoke restructure loop (old lines 67-78) is deleted in favor of
  this central version, and `build_ui.sh` (the dev script) gets the
  same logic at lines 56-60. Three independent restructure
  implementations → one source of truth.

- **`.litellm_ui_ready` marker preserved.** `Dockerfile.non_root:65`
  still touches `/var/lib/litellm/ui/.litellm_ui_ready`, which
  `proxy_server.py:_is_ui_pre_restructured()` checks at startup to
  short-circuit the runtime rewrite. Without this, the runtime would
  re-do the rewrite on every boot. The PR description explicitly calls
  out preserving it — good attention to runtime invariants.

- **`build_release_ui.sh` no longer auto-commits.** Old script ran the
  build and then `git add -f $destination_dir && git commit` on every
  invocation, which was the mechanism by which the bundle ended up
  checked in. New script (lines 4-15) just delegates to
  `docker/build_admin_ui.sh` with a comment explaining the behavior
  change. This is the right way to deprecate the auto-commit path.

- **Coverage matrix in the PR body.** The PR explicitly enumerates
  all 8 surfaces (5 Dockerfiles, PyPI publish, two CircleCI jobs) and
  states which ones now build via `build_admin_ui.sh` vs already did
  vs do their own inline build. That's the kind of audit table that
  makes follow-up bundle-removal review trivial.

- **PyPI publish step is correctly placed.** Step at
  `publish_to_pypi.yml:124-138` runs `setup-node@v5.0` + `npm ci` +
  `./docker/build_admin_ui.sh` *before* `uv build`, so the wheel/sdist
  picks up the freshly built `_experimental/out`. The
  `cache-dependency-path: ui/litellm-dashboard/package-lock.json` is
  correct (caches based on the actual lockfile that `npm ci` consumes,
  not the repo root). Pinning `setup-node` to a SHA
  (`a0853c24544627f65ddf259abe73b1d18a591444`) instead of a moving
  `v5` tag is the supply-chain-correct choice.

- **`Dockerfile.alpine` apk additions are minimal.** Adds `bash` to
  the apk install at line 17 (previously absent — needed because
  `build_admin_ui.sh` is `#!/bin/bash`). Without this the script
  would have failed at startup on Alpine. Catch confirms the author
  actually built the Alpine image.

## Nits / before-merge

1. **`Dockerfile.non_root` ordering: build before `chown`?** The new
   build step at `Dockerfile.non_root:62-63` runs as the build user
   already established earlier in the file. Confirm that the
   subsequent `cp -r /app/litellm/proxy/_experimental/out/.
   /var/lib/litellm/ui/` (line 67) still has the right ownership for
   the runtime non-root user — previously the bundle came in via
   `COPY . .` with `--chown` semantics; now it's generated in-image,
   which may or may not inherit the same ownership.

2. **CircleCI `ui_build` job not updated to use new script.** The
   coverage matrix says it still uses
   `ui/litellm-dashboard/build_ui.sh` (existing). That's fine — but
   `build_ui.sh` now also does the HTML restructure, so the CircleCI
   `ui_build` artifact (if anything downstream consumes it) will start
   shipping `foo/index.html` instead of `foo.html`. Worth a quick
   audit that no downstream test fixture pins the old layout.

3. **CRLF guard `sed -i 's/\r$//' docker/build_admin_ui.sh` is
   defensive but masks symptoms.** Both Dockerfiles strip CRLF before
   `chmod +x`. If any contributor commits the script with CRLF (e.g.,
   from Windows checkout with `core.autocrlf=true`), the build still
   works — but it also means a `.gitattributes` rule pinning the
   script to LF would be more robust. Not a blocker.

4. **No CircleCI link or "Branch creation CI run" link in the
   checklist.** PR template fields are unchecked. Maintainers will
   want at least one green CI run on this branch before merge,
   especially for the PyPI publish workflow change (which only fires
   on release tags, so it can't be tested by branch CI alone — a
   workflow_dispatch dry-run would help).

5. **The base branch is `litellm_internal_staging`.** This is an
   internal staging branch, not `main`. The PR will need a follow-up
   merge from `litellm_internal_staging → main`; whoever lands this
   should make sure the staging diff stays clean so that follow-up
   merge is mechanical.

## Risk

- **Low for the docker images** — the build is exercised on every
  image build, so a broken script would surface immediately.
- **Higher for PyPI publish** — the new step only runs on the publish
  workflow (release tag push). A dry-run via `workflow_dispatch` or a
  test tag is the right gate before relying on this for the next
  release.
- **The follow-up bundle-removal PR is where things get scary** — but
  this PR is the right shape for that follow-up to be safe.

## Verdict

`merge-after-nits` — the structural change is correct and the
fact-pattern (every artifact path now builds from source) is exactly
what the bundle-removal needs. Nits are: get a green PyPI
workflow_dispatch run, confirm `Dockerfile.non_root` ownership of the
generated bundle, and audit downstream consumers of the CircleCI
`ui_build` artifact for the new HTML layout.
