---
pr: 24520
repo: sst/opencode
sha: 56203dc1cd8654e6dc0f8049d556ed73013b7ad3
verdict: merge-after-nits
date: 2026-04-27
---

# sst/opencode #24520 — feat(docker) build ubuntu image alongside alpine

- **Head SHA**: `56203dc1cd8654e6dc0f8049d556ed73013b7ad3`
- **Author**: randommm
- **Size**: small (Dockerfile + publish.ts)

## Summary
Parameterizes the Dockerfile via two new build-args (`BASE_IMAGE`, `DIST_SUFFIX`) so the same recipe can produce alpine (musl) and ubuntu (glibc) images, and `publish.ts` now does two `docker buildx build` invocations to push both. Also adds a `:latest` tag to the alpine variant.

## Specific findings
- `packages/opencode/Dockerfile:1-2` — `ARG BASE_IMAGE=alpine:3.21` pins alpine *and* bumps it from the previous floating `alpine` tag to `alpine:3.21`. Pinning is good, but call it out in the PR body — it's a behavior change separate from the ubuntu work.
- `packages/opencode/Dockerfile:9-21` — single `RUN` block uses `if [ -f /etc/alpine-release ]` to dispatch between `apk` and `apt-get`. Cheap and effective. Two issues:
  1. The detection is *file-presence-based*, so any future base image that happens to ship `/etc/alpine-release` (debian-with-compat, rare but possible in CI test images) would mis-route to `apk`. A more robust check is `command -v apk >/dev/null 2>&1`.
  2. On the ubuntu branch, `DEBIAN_FRONTEND=noninteractive` is set inline but `apt-get update` runs first without it. Move the env to the front of the line: `DEBIAN_FRONTEND=noninteractive apt-get update && DEBIAN_FRONTEND=noninteractive apt-get install ...`. Avoids tzdata/console-data prompts if dependencies pull them in transitively in the future.
- `packages/opencode/Dockerfile:13-17` — `case "$arch" in amd64) arch=x64 ;; arm64) arch=arm64 ;; *) exit 1 ;;` — the `exit 1` happens inside a `RUN` shell *after* `cp` would have failed anyway, but more importantly there's no error message. `*) echo "unsupported arch: $arch" >&2; exit 1 ;;` saves debug time.
- `packages/opencode/Dockerfile:18-19` — `cp ... /usr/local/bin/opencode && chmod +x` — the previous Dockerfile used `COPY` which preserves perms from the dist tarball. The dist tarball already has `+x`, so the explicit `chmod` is defensive but harmless.
- `packages/opencode/Dockerfile` overall — `COPY dist /tmp/dist` copies the *entire* dist directory into every build, then only one subdir is used. Increases image-build cache invalidation: any change to any platform's binary busts the cache for both builds. Consider `COPY dist/opencode-linux-${TARGETARCH}* /tmp/dist/` if buildx's `TARGETARCH` is available at COPY time (it is, with `# syntax=docker/dockerfile:1.4`+).
- `packages/opencode/script/publish.ts:65-67` — extracted `buildImage()` helper is clean. Tag list for the alpine build now includes `${image}:latest`, which was *not* previously published per the diff. New behavior: every release also moves `:latest`. Fine if intentional, but flag in the changelog — users who pinned to `:latest` for "stable channel" will start tracking dev/beta if `Script.channel` is ever non-prod.
- `packages/opencode/script/publish.ts:79-84` — ubuntu build only emits `${image}:ubuntu` — no version tag (`${image}:${version}-ubuntu`), no channel tag. Fine for "always-latest ubuntu" but means users can't pin a specific version of the ubuntu image. Add `${image}:${version}-ubuntu` for parity with alpine.

## Risk
Low. Two-image publish doubles build time but the surfaces are independent.

## Verdict
**merge-after-nits** — fix `apk`-detection to use `command -v`, document the `:latest` move and the alpine pin, and add a versioned ubuntu tag.
