# QwenLM/qwen-code #3683 — Upgrade GitHub Actions to latest versions

- SHA: `4c7994dd51b119ba4053e7d7536192fb96823e77`
- State: OPEN, +21/-21 across 11 workflow files
- Scope: bump 14 distinct actions to latest pinned SHAs

## Summary

Bulk maintenance bump of 14 GitHub Actions across 11 workflows. Key major-version moves:

- `docker/setup-qemu-action` v3 → v4
- `docker/setup-buildx-action` v3 → v4
- `docker/metadata-action` v5 → v6
- `docker/login-action` v3 → v4
- `docker/build-push-action` v6 → v7
- `astral-sh/setup-uv` v6 → v8.1.0
- `actions/configure-pages` v5 → v6
- `actions/upload-pages-artifact` v3 → v5
- `actions/deploy-pages` v4 → v5

Plus minor patch bumps (`actions/create-github-app-token`, `dorny/test-reporter`, `google-github-actions/run-gemini-cli`, `qwenlm/qwen-code-action`, `thollander/actions-comment-pull-request`).

## Notes

- Pinning style is mixed: some bumps land as `@v4` floating tags (e.g., `docker/setup-qemu-action@v4` in `build-and-publish-image.yml:70`), others as full SHAs with version comments (e.g., `actions/create-github-app-token@1b10c78c...  # v3.1.1`). The pre-existing convention used SHA pins with `# ratchet:...@v3` comments — the diff partly abandons that. Either keep ratchet-style SHA pins for everything or move everything to floating tags; the current mixed state will fight any SHA-pin enforcement check.
- `build-and-publish-image.yml:70-100` — five Docker actions cross a major boundary. Worth a CI dry-run before merge to confirm no breaking-change input rename (e.g., `docker/build-push-action` v6→v7 changed default cache behavior in some releases).
- `terminal-bench.yml:33` — `astral-sh/setup-uv` v6 → v8.1.0 skips v7 entirely. Their v7 release notes flagged a behavior change around lockfile handling; v8 might re-stabilize but worth confirming `uv` flow still works.
- `e2e.yml:51` and `release.yml:261` — both bump `docker/setup-buildx-action` to v4 but use SHA `4d04d5d9...` whereas other files use floating `@v4`. Consistency.
- The PR body's table is exhaustive and helpful — every bump has an upstream compare link.
- PR author explicitly suggests "running the workflows on a branch before merging" — agree, this is the right gate, especially for the Docker stack.

## Verdict

`merge-after-nits` — useful hygiene, but: (1) reconcile SHA-pin vs floating-tag convention, (2) run `build-and-publish-image.yml` on a branch first to validate the Docker v6→v7 hop, (3) confirm `terminal-bench.yml` still passes with `setup-uv` v8.
