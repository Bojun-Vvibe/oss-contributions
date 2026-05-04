# BerriAI/litellm #27135 — Stop tracking the pre-built Admin UI bundle in litellm/proxy/_experimental/out/

- URL: https://github.com/BerriAI/litellm/pull/27135
- Head SHA: `d160461dc6485d2c93aa0b13da412115dcbf35d9`
- Author: mateo-berri
- Size: +92 / -9909 across many files

## Comments

1. `.gitignore` — Adding `litellm/proxy/_experimental/out/` (presumably) to gitignore is the right call; verify the path glob also covers the nested `_next/static/chunks/*.js` so future build artefacts can't sneak back in. Please paste the exact gitignore line so reviewers can double-check.
2. `docker/Dockerfile.alpine` and `docker/Dockerfile.non_root` — Both Dockerfiles must now produce the bundle at build time. Confirm `docker/build_admin_ui.sh` is invoked in *both* and that the resulting `out/` directory survives any later `COPY --from=builder` layer reset. A broken Docker build here ships a UI-less admin panel.
3. `docker/build_admin_ui.sh` — New script: ensure it pins a Node version (the rest of the repo standardises on Node 20). Without `engines`-pin you'll get nondeterministic chunk hashes across CI nodes.
4. `.github/workflows/publish_to_pypi.yml` — pypi sdist must include the freshly built `out/` directory; if `MANIFEST.in` doesn't include it, `pip install litellm[proxy]` users get a broken UI. Add an explicit `MANIFEST.in` line and a smoke test that imports the proxy admin route.
5. The deletion list shows ~9.9k lines of pre-built Next.js chunks. Once merged, history pulls become noticeably lighter; good. But also confirm no test or runtime path does `os.path.exists("litellm/proxy/_experimental/out/index.html")` as a heuristic — would silently start failing.
6. PR description should call out the version of litellm where the in-tree bundle is removed so downstream packagers (debian, conda) can update build pipelines.

## Verdict

`needs-discussion`

## Reasoning

The intent — stop committing minified JS into the repo — is correct and overdue. But this is a packaging change with real downstream impact: every distributor that consumes litellm via `pip install` (no Docker) needs the UI bundle to either ship in the wheel or be auto-built at install time, neither of which is obvious from the diff slice visible here. Wheel/sdist coverage and the Dockerfile build paths must be verified end-to-end (build, install, hit `/ui`) before this lands, otherwise downstream users will report "blank admin UI" on first upgrade.
