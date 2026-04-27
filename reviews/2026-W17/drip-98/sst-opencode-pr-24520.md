# sst/opencode#24520 — feat(docker) build ubuntu image alongside alpine

- **Head**: `56203dc1cd8654e6dc0f8049d556ed73013b7ad3`
- **Size**: +37/-13 across 2 files
- **Verdict**: `merge-after-nits`

## Context

Reporter hit a real ROCm-PyTorch-on-musl incompatibility with the existing `alpine`-based `packages/opencode/Dockerfile`. Rather than swap base images and break the published image's churn-free identity, the PR parameterizes the base via `ARG BASE_IMAGE=alpine:3.21` + `ARG DIST_SUFFIX=-musl` and ships **two parallel images** under the same `ghcr.io/anomalyco/opencode` registry: alpine (default tags `:version`, `:channel`, **`:latest`**) and Ubuntu 24.04 (single tag `:ubuntu`).

## Design analysis

The Dockerfile rewrite at `packages/opencode/Dockerfile:1-28` collapses the previous `build-amd64`/`build-arm64` per-target stages into a single stage with conditional package install:

```dockerfile
RUN if [ -f /etc/alpine-release ]; then \
      apk add --no-cache libgcc libstdc++ ripgrep; \
    else \
      apt-get update; \
      DEBIAN_FRONTEND=noninteractive apt-get install -y --no-install-recommends libgcc-s1 libstdc++6 ripgrep; \
      rm -rf /var/lib/apt/lists/*; \
    fi && \
    arch="$TARGETARCH" && \
    case "$arch" in \
      amd64) arch=x64 ;; \
      arm64) arch=arm64 ;; \
      *) exit 1 ;; \
    esac && \
    cp "/tmp/dist/opencode-linux-${arch}${DIST_SUFFIX}/bin/opencode" /usr/local/bin/opencode && \
    chmod +x /usr/local/bin/opencode
```

Three things to note:

1. The `/etc/alpine-release` probe is the right idiom (more robust than `$BASE_IMAGE` substring matching).
2. `--no-cache` on the apk side and `rm -rf /var/lib/apt/lists/*` on the apt side both keep image bloat in check.
3. `COPY dist /tmp/dist` at `:11` now copies *the whole `dist/` tree* into the build context vs the old `COPY dist/opencode-linux-x64-baseline-musl/bin/opencode …` projection. That's the right move for arch-conditional pickup but inflates the build context and final layer with at least the unused arch's binary unless the publish script trims first.
4. `apk add libgcc libstdc++ ripgrep` had no `--no-cache` in the original; the new version does, so alpine image gets ~3-5 MB smaller as a free side-effect.

`script/publish.ts:64-90` factors a `buildImage(tags, buildArgs)` helper and calls it twice, so the new Ubuntu build adds one full `docker buildx build --platform linux/amd64,linux/arm64 --push` invocation per release — non-trivial CI time cost (~2-5 min on top of an already-multi-platform alpine push).

## Nits / concerns

1. **`:latest` tag now points at alpine.** The old code didn't push `:latest` — only `:version` and `:channel`. The new alpine `buildImage` call at `:74-80` adds `${image}:latest` to the tag list. Worth maintainer signoff (this is a contract change on the registry), and the PR description doesn't flag it.
2. **Verification gap.** PR body admits "I can't verify this one exactly because the all CI don't run on forks" and ticks "I have tested my changes locally" anyway with the caveat "is not applicable". The `--build-arg` matrix and arch-mapping `case` block in the new Dockerfile have enough fork points that an author-side `docker buildx build --build-arg BASE_IMAGE=ubuntu:24.04 --build-arg DIST_SUFFIX= --platform linux/amd64,linux/arm64 .` smoke would close the loop.
3. **`exit 1` swallows arch info.** `*) exit 1 ;;` at `:21` should be `*) echo "Unsupported TARGETARCH: $TARGETARCH" >&2; exit 1 ;;` — when this fires in CI six months from now nobody will remember why.
4. **Glibc-binary tag/source binding.** The Ubuntu image builds with `DIST_SUFFIX=""` so it pulls `dist/opencode-linux-x64/bin/opencode` (the default glibc binary, not the `-musl` variant). That's correct but should be documented in the Dockerfile header — readers will rightly wonder why "alpine:3.21 + musl binary" and "ubuntu:24.04 + glibc binary" share a build path.
5. **No CI matrix update.** If the published-image release pipeline has a `docker manifest inspect` smoke or version-readback step, it now needs to cover `:ubuntu` too; not in this diff.

## What I learned

The `BASE_IMAGE` + `DIST_SUFFIX` ARG pair is the right fan-out shape for "same code, two libc surfaces" Docker images. The big risk in shipping a second variant under the same registry is *tag drift over time* — `:latest` and `:ubuntu` will quietly disagree on what `opencode --version` reports if a future release forgets to push one of them. A CI guard that compares `docker run --rm $image:latest --version` and `docker run --rm $image:ubuntu --version` after each release would prevent that whole class of bug at near-zero cost.
