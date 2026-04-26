# Review — sst/opencode#24520: feat(docker) build ubuntu image alongside alpine

- **Repo:** sst/opencode
- **PR:** [#24520](https://github.com/sst/opencode/pull/24520)
- **Author:** randommm (Marco Inacio)
- **Head SHA:** `56203dc1cd8654e6dc0f8049d556ed73013b7ad3`
- **Size:** +37 / −13 across 2 files (`packages/opencode/Dockerfile`, `packages/opencode/script/publish.ts`)
- **Verdict:** `merge-after-nits`

## Summary

Replaces the existing two-stage alpine-only Dockerfile with a single-stage parametric build that switches on `ARG BASE_IMAGE` (default `alpine:3.21`) and `ARG DIST_SUFFIX` (default `-musl`), detecting the runtime via `[ -f /etc/alpine-release ]` to pick `apk`/`apt-get`. `publish.ts:65-86` is rewritten to call a new `buildImage(tags, buildArgs)` helper twice — once for the alpine `:version`/`:channel`/`:latest` tags with `BASE_IMAGE=alpine:3.21 DIST_SUFFIX=-musl`, once for `:ubuntu` with `BASE_IMAGE=ubuntu:24.04 DIST_SUFFIX=` (i.e. the glibc artifact). Motivated by PyTorch ROCm not running cleanly on musl.

## Technical assessment

The Dockerfile collapse is correct. The old `FROM base AS build-amd64` / `FROM build-${TARGETARCH}` indirection at the prior `Dockerfile:9-17` was just a way to pick the binary path per arch; replacing it with an inline `case "$arch" in amd64) arch=x64 ;; arm64) arch=arm64 ;;` at `Dockerfile:21-25` plus a single `cp` is a real readability win and preserves the same multi-arch buildx semantics. The `[ -f /etc/alpine-release ]` runtime detection at `Dockerfile:14-19` is a standard idiom and works for both alpine 3.21 and ubuntu 24.04 — `ubuntu:24.04` ships `libgcc-s1`/`libstdc++6`/`ripgrep` in `apt`'s default sources so the install is single-shot.

`publish.ts:67-86` is the riskier surface. The new `buildImage` closure is passed `tagFlags` flattened from `["-t", tag]` pairs and a positional `buildArgs` array, then spread into `docker buildx build --platform ${platforms} ${tagFlags} ${buildArgs} --push .`. Bun's `$` template with array interpolation does pass each element as a separate argv, so `--build-arg BASE_IMAGE=alpine:3.21 --build-arg DIST_SUFFIX=-musl` will land correctly — but the change also adds `${image}:latest` to the alpine-tag set, which the prior code did not push. That's a behavior change beyond "build ubuntu alongside alpine" and worth calling out in the PR body.

## Nits worth addressing pre-merge

1. **`:latest` tag is a behavior change.** At `publish.ts:75` the alpine push now includes `${image}:latest`. The previous code only pushed `:${version}` and `:${Script.channel}`. If the channel is `dev` or `nightly`, `:latest` will get overwritten by every nightly. Either (a) drop `:latest` from this PR and ship as a follow-up gated on `Script.channel === "stable"`, or (b) explicitly gate it: `Script.channel === "stable" ? [..., ":latest"] : [...]`.

2. **Sequential pushes, not parallel.** `await buildImage(alpine)` then `await buildImage(ubuntu)` doubles publish wall-clock for what could be `Promise.all([alpineBuild, ubuntuBuild])`. Buildx with two distinct tag sets and two distinct base images can run concurrently against the same buildx builder. Not blocking, but a 2× speedup is on the table for free.

3. **`DIST_SUFFIX=""` magic.** The non-musl artifact lives at `dist/opencode-linux-${arch}/bin/opencode` (no suffix). `DIST_SUFFIX=` works because `${arch}${DIST_SUFFIX}` collapses to `${arch}`, but a comment near `Dockerfile:27` clarifying that empty string = glibc artifact would save the next reader 30 seconds.

4. **`apt-get install` not pinned.** `Dockerfile:17` installs `libgcc-s1 libstdc++6 ripgrep` from ubuntu's default channel. Reproducibility-conscious users will want to pin these or at least pin the ubuntu base image to a digest rather than a floating `:24.04` tag. Future improvement.

5. **CI verification gap.** PR body notes "the all CI don't run on forks" so the ubuntu path is not actually built by CI on this PR. Before merge, a maintainer should kick off a manual buildx run from a branch in the source repo to confirm `ubuntu:24.04` actually finds `ripgrep` in the default apt sources (it does on jammy and noble, but worth verifying against the real artifact tree).

## Verdict rationale

`merge-after-nits` because the Dockerfile parameterization is clean and the use case (PyTorch ROCm needing glibc) is real, but the silent `:latest` tag addition and the lack of fork-CI verification mean a maintainer should sanity-check both the buildx run and the publish-side tag policy before this lands. None of the nits require a rewrite.
