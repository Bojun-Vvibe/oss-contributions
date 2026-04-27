# QwenLM/qwen-code#3683 — Upgrade GitHub Actions to latest versions

- **PR**: https://github.com/QwenLM/qwen-code/pull/3683
- **Author**: salmanmkc (Salman Chishti)
- **Head SHA**: `4c7994dd51b119ba4053e7d7536192fb96823e77`
- **Verdict**: `merge-after-nits`

## Context

The qwen-code repository pins all third-party GitHub Actions to specific commit SHAs with a trailing `# ratchet:owner/repo@vN` comment — the standard `ratchet`-tool pattern that lets a bot detect when a major version has shipped and propose a SHA bump. This PR is a bulk-update sweep across roughly a dozen workflow files: `actions/configure-pages` v5→v6, `actions/upload-pages-artifact` v3→v5, `actions/deploy-pages` v4→v5, `actions/create-github-app-token` v2→v3.1.1, `docker/setup-qemu-action` v3→v4, `docker/setup-buildx-action` v3→v4, `docker/metadata-action` v5→v6, `docker/login-action` v3→v4.1.0, `docker/build-push-action` v6→v7, `dorny/test-reporter` v2→v3.0.0, `thollander/actions-comment-pull-request` v3→v3.0.1, `google-github-actions/run-gemini-cli` v0→v0.1.22.

## What it changes

Mechanical SHA bumps across 12+ workflow files. The pattern is consistent throughout: each `uses: 'owner/repo@<old-sha>' # ratchet:owner/repo@vN` line is rewritten to `uses: 'owner/repo@<new-sha>'  # vN.M.P` (note the formatting drift — see nit 1 below). Spot-checking a few of the bumps against upstream changelogs:

- **`docker/setup-buildx-action` v3 → v4** is a major bump that drops Node 16 runner support and requires Node 20+ — qwen-code's CI runs on `ubuntu-latest` which has Node 20+ since mid-2024, so safe.
- **`docker/build-push-action` v6 → v7** is a major bump that changes the default Buildx version requirement and the cache-export shape — should be safe for the existing build matrix but worth a single test run before merge.
- **`actions/upload-pages-artifact` v3 → v5** skips v4 entirely; v4 introduced the `id-token: write` permission requirement for some flows, v5 stabilized it. The repository's `docs-page-action.yml` at `:36` uses this action — needs a check that the calling workflow has `permissions: id-token: write` set.
- **`google-github-actions/run-gemini-cli` v0 → v0.1.22** is a patch bump within the v0 line, low risk.
- **`actions/create-github-app-token` v2 → v3.1.1** is a major bump — v3 changed the input schema for some less-common knobs; the diff at `:75-78` and `:155-158` shows the existing `app-id` / `private-key` input shape, which is unchanged across v2 and v3, so the call sites are compatible.

## Design analysis

The right cadence for this kind of sweep is "all the ratchet-tracked actions in one PR" exactly as done here — splitting into one-PR-per-action would 12x the review overhead with no safety gain because the actions are independent of each other. The SHA-pinning convention (`owner/repo@<full-40-char-SHA>`) is preserved throughout, which is the security-relevant invariant — a `uses: 'owner/repo@v4'` would silently float to whatever the maintainer tags as `v4` next, and the explicit SHA is the supply-chain mitigation. Downside: the diff is hard to audit for "did the SHA actually correspond to the claimed version?" without checking each one against the upstream tag — which is exactly what the `ratchet` tool is *supposed* to do automatically, but a reviewer should still spot-check 2-3 of the major bumps.

## Concerns / nits

1. **Comment-format drift** — the existing convention is `# ratchet:owner/repo@vN` (single hash, single space, the literal string `ratchet:`). The new lines use `  # vN.M.P` (double space, no `ratchet:` prefix, exact version string). This breaks the ratchet-tool's ability to parse the file and propose future bumps — the next time `actions/checkout@v5` ships v6, the tool won't know to bump `actions/configure-pages` because its line no longer matches the parser regex. **This is the only blocking nit.** Either preserve the `# ratchet:owner/repo@vN` format on the bumped lines, or update the project's ratchet config to also recognize the new format. Inconsistency is the worst outcome — half the lines parse, half don't.

2. **No CI run referenced in the PR description** — for a sweep that includes two major-version bumps on Docker actions (`setup-buildx-action` v3→v4, `build-push-action` v6→v7), a successful end-to-end build-and-publish run on a test branch would close the regression-detection gap. The other bumps (Pages, run-gemini-cli, dorny/test-reporter) are lower-risk patch/minor bumps where ratchet's own version-resolution proves the SHA→version mapping.

3. **`actions/deploy-pages` v4 → v5** at `:115-116` — v5 requires `permissions: pages: write, id-token: write` on the calling job. The diff doesn't show the surrounding `permissions:` block; the reviewer should verify `docs-page-action.yml`'s deploy job already has both permissions set (it almost certainly does since v4 also required `pages: write`, but `id-token: write` might be new).

4. **`thollander/actions-comment-pull-request` v3 → v3.0.1** at `:9-10` — the diff shows a SHA bump with the comment `# v3.0.1`. v3.0.0 of this action introduced a breaking change to the `message` input shape (now requires the `message:` key instead of accepting the message body directly in some configurations). The call site at `post-coverage-comment/action.yml` should be re-checked for the new input shape. Worth a CI run.

5. **Mixed comment hygiene** — some bumped lines have `# v4.0.0`, some have `# v0.1.22`, some have `# v3.1.1`. The version-string format is inconsistent (some use full semver, some use partial). Not blocking but worth normalizing.

## Risks

Medium. Two major Docker-action bumps that change runtime requirements, one Pages action (`upload-pages-artifact`) that skips a major version. All three need a successful CI run to verify, and the comment-format drift (nit 1) breaks future ratchet automation if not fixed.

## Suggestions

Block on nit 1 (comment-format consistency — either preserve `# ratchet:owner/repo@vN` or update ratchet config) and ask for a successful end-to-end CI run on a test branch covering at least the build-and-publish-image, e2e, and docs-page-action workflows. The other nits are nice-to-have.

## What I learned

The `# ratchet:owner/repo@vN` comment convention is load-bearing for any project using the ratchet tool — it's not just human documentation, it's the parser anchor for "is this action up to date?" detection. Any bulk SHA-bump PR that doesn't preserve the comment format silently turns off the automation that prompted the PR in the first place. Worth grepping for `# ratchet:` on any future SHA-pinned-actions PR to make sure the comment shape survives.
