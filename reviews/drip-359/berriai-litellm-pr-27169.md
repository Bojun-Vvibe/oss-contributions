# BerriAI/litellm PR #27169 — [Perf] CI: Skip Redundant Playwright Apt Install in E2E UI Job

- Repo: `BerriAI/litellm`
- PR: #27169
- Head SHA: `19ad964c4a8b6eba7d30e3de720b5e8391813ba7`
- Author: `yuneng-berri`
- Updated: 2026-05-05T04:21:54Z
- Verdict: **merge-after-nits**

## What it does

CI-only perf change in `.circleci/config.yml:2138-2157` for the E2E UI Playwright job. Three coordinated edits:

1. **`npx playwright install chromium --with-deps` → `npx playwright install chromium`** (drops `--with-deps`).
2. **Cache key bump** `ui-e2e-node-deps-v1-...` → `ui-e2e-node-deps-v2-...` (both restore_cache and save_cache) so the new cache layout is treated as a new cache namespace.
3. **Adds `~/.cache/ms-playwright` to the `paths:` of `save_cache`** so the downloaded Chromium browser binary itself is now cached alongside `node_modules`.

The PR adds a useful inline justification comment at lines 2143-2147:
```
# The cimg/python:3.12-browsers image already ships the Chromium system
# libraries Playwright needs (libnss3, libatk-bridge2.0-0, libcups2, etc.).
# `--with-deps` triggers a redundant apt-get update + install that adds
# 5-10 minutes to the job and frequently stalls on flaky Ubuntu mirrors,
# so we install just the browser binary.
```

## Strengths

- **Correct premise.** The CircleCI `cimg/python:3.12-browsers` convenience image is explicitly the "browsers preinstalled" variant — it ships the X11/font/audio system libraries Playwright needs. Running `--with-deps` on top re-runs `apt-get update && apt-get install -y ...` for libraries that are already present, which is the documented anti-pattern for the `*-browsers` cimg variants. Skipping it should save the documented 5-10 minutes per E2E run.
- **Cache-key bump is correct hygiene.** Adding a new path to `save_cache` while keeping the old key would mean the cache hit on the old key would restore *without* `~/.cache/ms-playwright`, masking the perf win on every cache hit until the lockfile changes. The `v1 → v2` bump forces a one-time cache miss that populates the new layout.
- **Self-documenting comment.** Future contributors who see "why is `--with-deps` missing?" get the answer in-line. Reduces the chance of a well-meaning revert.

## Nits / concerns

1. **No verification that the base image actually carries every transitive dep Chromium needs.** The comment lists `libnss3, libatk-bridge2.0-0, libcups2` — these are the obvious ones, but Playwright's `--with-deps` set is larger (libdrm2, libgbm1, libxkbcommon0, libxcomposite1, libxdamage1, libxfixes3, libxrandr2, libpango-1.0-0, libcairo2, libasound2, libatspi2.0-0, fonts-liberation, etc.). If the `cimg/python:3.12-browsers` tag ever drifts to a newer Ubuntu base that drops one of these, the E2E job will fail with a confusing `error while loading shared libraries: libXXX.so.N` message at *test runtime*, not at install time. Worth either:
   - adding a `playwright check-system-deps` step (or `ldd $(which chromium) | grep "not found"`) as a fast-fail gate, **or**
   - pinning the `cimg/python:3.12-browsers` tag with a digest in the executor block so a base-image bump is an explicit PR.

2. **Cache key only tracks `package-lock.json`.** The new cache also stores the Chromium binary at `~/.cache/ms-playwright`, but the cache key checksum is still over `ui/litellm-dashboard/package-lock.json` only. If `package-lock.json` doesn't change but the Playwright **version** in `package.json` is bumped (e.g. via a `^1.42.0 → ^1.45.0` resolver bump that doesn't touch the lockfile root), the cached browser binary will be the wrong version for the installed Playwright NPM package. In practice `npm ci` rewrites the lockfile so this is rare, but a safer key would be `{{ checksum "ui/litellm-dashboard/package-lock.json" }}-{{ arch }}` or include the Playwright version explicitly.

3. **No measured before/after timing in the PR description.** The "5-10 minutes" claim in the comment is reasonable for an apt-get-update on a flaky mirror, but a single CircleCI run-link comparison (before-PR job duration vs. after-PR job duration) would convert this from a plausible perf claim to a verified one and discourage future "let me put `--with-deps` back to be safe" reverts.

4. **Cache restore key list is single-key.** `keys:` is given a single fully-qualified key with `{{ checksum ... }}`. A fallback like `ui-e2e-node-deps-v2-` (no checksum) as a second entry would let a partial cache hit speed up the *next* run after a lockfile bump too. Optional.

## Verdict
**merge-after-nits** — premise is correct (`*-browsers` cimg variants are designed to skip `--with-deps`), the cache-key bump is the right move, and the inline comment will save the next contributor a debug session. Add a `check-system-deps` fast-fail step or pin the executor image digest before considering the perf optimization durable.
