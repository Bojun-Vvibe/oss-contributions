---
pr: https://github.com/google-gemini/gemini-cli/pull/26016
sha: fefde38b
diff: +5/-3
state: OPEN
---

## Summary

Three-line documentation hygiene fix targeting two distinct broken-link classes in the contributor guide: (1) a root-relative `/docs/assets/...` image reference inside `CONTRIBUTING.md` that breaks when the file is rendered outside the docs site, and (2) the `docs/CONTRIBUTING.md` symlink-forwarder which renders as a broken link on GitHub's web view because GH doesn't follow same-repo markdown symlinks.

## Specific observations

- `CONTRIBUTING.md:416` — image path changes from `![](/docs/assets/connected_devtools.png)` (root-anchored, only resolves on the docs site under a specific base path) to `![Connected React DevTools](docs/assets/connected_devtools.png)` (relative, resolves correctly both on github.com and on the docs site). Also adds alt text — accessibility win, free.
- `docs/CONTRIBUTING.md` — the file changes mode from a symlink (`120000`) pointing at `../CONTRIBUTING.md` to a regular markdown file containing a single forwarding link. This is the correct fix for the GitHub-renders-symlinks-as-broken-links issue; the alternative (delete the file and rely on the root copy) would break any external doc site or third-party tool that has hardcoded the `docs/CONTRIBUTING.md` path. Three-line forwarding stub is the minimal contract.
- `docs/index.md:127` — `/docs/contributing` (a route that does not exist in the docs site routing per the bug being fixed) becomes `../CONTRIBUTING.md` (relative path that works in both GitHub web view and the static-site renderer). Slight asymmetry: the other docs links on the same page (`./integration-tests.md`, `./issue-and-pr-automation.md`) use `./` prefix; this one uses `../` because `CONTRIBUTING.md` lives at repo root not under `docs/`. Correct.

## Verdict

`merge-as-is` — pure docs fix, three surgical edits, no behavioral risk, fixes a real broken-link class on the project's primary contributor onboarding surface. The symlink-to-stub conversion is the right call given GitHub's known rendering behavior on cross-directory markdown symlinks.
