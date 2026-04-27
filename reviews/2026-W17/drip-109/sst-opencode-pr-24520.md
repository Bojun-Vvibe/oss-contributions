# sst/opencode #24520 — feat(docker) build ubuntu image alongside alpine

- **Repo**: sst/opencode
- **PR**: #24520
- **Author**: randommm
- **Head SHA**: 56203dc1cd8654e6dc0f8049d556ed73013b7ad3
- **Size**: +37 / −13 across two files (`packages/opencode/Dockerfile`, `packages/opencode/script/publish.ts`).
- **Closes**: #24521

## What it changes

Two coordinated edits to publish an Ubuntu-based Docker image alongside the
existing Alpine one, motivated by musl/glibc-only workloads (the PR body cites
PyTorch ROCm failing inside the alpine image because of musl):

1. `packages/opencode/Dockerfile:1-28` is restructured from a three-stage
   `FROM base AS build-amd64` / `build-arm64` / `FROM build-${TARGETARCH}`
   layout into a single base-arg-driven stage. The new file:
   - Adds `ARG BASE_IMAGE=alpine:3.21` and `ARG DIST_SUFFIX=-musl` so the
     same Dockerfile builds either alpine-musl or ubuntu-glibc images.
   - Replaces the unconditional `RUN apk add libgcc libstdc++ ripgrep` with a
     branch on `[ -f /etc/alpine-release ]`: alpine path keeps `apk add
     --no-cache libgcc libstdc++ ripgrep`; non-alpine path runs `apt-get
     update && DEBIAN_FRONTEND=noninteractive apt-get install -y
     --no-install-recommends libgcc-s1 libstdc++6 ripgrep` followed by
     `rm -rf /var/lib/apt/lists/*`.
   - Maps `TARGETARCH` (`amd64`/`arm64`) to the dist-tarball arch suffix
     (`x64`/`arm64`) inside a `case "$arch" in` and copies
     `/tmp/dist/opencode-linux-${arch}${DIST_SUFFIX}/bin/opencode` to
     `/usr/local/bin/opencode`, then `chmod +x`.
2. `packages/opencode/script/publish.ts:63-87` extracts the buildx invocation
   into a `buildImage(tags, buildArgs)` helper, then calls it twice in the
   `!Script.preview` branch:
   - Alpine: tags `${image}:${version}`, `${image}:${Script.channel}`,
     **and a new `${image}:latest`** with `BASE_IMAGE=alpine:3.21` /
     `DIST_SUFFIX=-musl`.
   - Ubuntu: single tag `${image}:ubuntu` with `BASE_IMAGE=ubuntu:24.04` /
     `DIST_SUFFIX=` (empty, so the path resolves to the glibc tarball).

## Strengths

- The Dockerfile collapse (3 stages → 1) is the correct refactor; the previous
  `FROM build-${TARGETARCH}` indirection only existed to switch source paths
  and is fully replaced by a `case` statement plus the `DIST_SUFFIX` arg.
  No functional drift on the alpine path: the previous `apk add libgcc
  libstdc++ ripgrep` is byte-equivalent to the new `apk add --no-cache
  libgcc libstdc++ ripgrep` (the `--no-cache` flag is a strict
  size/correctness improvement — alpine's apk caches were already wiped at
  build time by buildx).
- The `apt-get` choices are right for a minimal runtime: `libgcc-s1` +
  `libstdc++6` are the matching glibc runtime libs, `--no-install-recommends`
  prevents pulling docs/manpages, and the `rm -rf /var/lib/apt/lists/*` cleanup
  inside the same `RUN` layer keeps the image from carrying ~30 MB of apt
  index files.
- `RUN opencode --version` at `Dockerfile:28` is preserved as the post-copy
  smoke test, so a broken arch mapping or a missing tarball would fail the
  build immediately rather than producing a silently broken image.
- `publish.ts:66-69` correctly hides the per-image differences behind one
  helper, so the two `await buildImage(...)` calls remain readable and the
  `--push` semantics, `--platform linux/amd64,linux/arm64` cross-build, and
  tag-flag flattening are not duplicated.

## Concerns / nits

- **`${image}:latest` is added silently to the alpine call** at
  `publish.ts:74`. The previous code produced two tags
  (`${version}` + `${Script.channel}`); the new code adds a third. That's
  almost certainly desirable (most users `docker pull foo` expect `:latest`
  to resolve), but it's a behavior change that isn't called out in the PR
  body and could surprise downstream `:dev` / `:nightly` channels where
  `latest` should track stable. Worth a one-line PR-body note plus a guard:
  `${Script.channel === "stable" ? [\`${image}:latest\`] : []}`.
- **The ubuntu image is only published with the `:ubuntu` tag**, with no
  `${image}:${version}-ubuntu` or `${image}:${Script.channel}-ubuntu`
  variant. Operators who want to pin to a specific opencode version on the
  ubuntu base have no way to do so without manually digesting the image. A
  symmetric tagging scheme (`[\`${image}:${version}-ubuntu\`,
  \`${image}:${Script.channel}-ubuntu\`, \`${image}:ubuntu\`]`) would close
  this and cost two array entries.
- **The non-alpine branch has no failure mode for unrecognized base images.**
  Today the only options exercised are `alpine:3.21` and `ubuntu:24.04`, but
  a future `BASE_IMAGE=debian:bookworm-slim` would silently take the
  `apt-get` path with the wrong package names (`libgcc-s1` is correct on
  Debian/Ubuntu, but `libstdc++6` package naming has drifted on some bases).
  Either documenting "supported BASE_IMAGE values: alpine:* and ubuntu:24.04"
  in a Dockerfile comment, or detecting `[ -f /etc/debian_version ]`
  explicitly, would prevent quiet breakage.
- **No CI coverage.** The author notes upfront that "all CI don't run on
  forks" so the ubuntu build was not exercised. A maintainer-side smoke
  job that runs `docker buildx build --build-arg BASE_IMAGE=ubuntu:24.04
  --build-arg DIST_SUFFIX= --target=base .` against a sample tarball would
  catch breakage at PR time rather than at the next release.
- The `COPY dist /tmp/dist` at `Dockerfile:13` copies the entire `dist/`
  directory into the image's intermediate filesystem before selecting one
  binary, then leaves `/tmp/dist` in the final image (the `cp` doesn't
  delete the source). For a multi-arch dist tree with both x64 and arm64
  musl + glibc tarballs, that's tens of MB of dead weight per layer. A
  `COPY --from=...` selective stage or a trailing `&& rm -rf /tmp/dist` in
  the same `RUN` would shed the bloat.

## Verdict

**merge-after-nits.** The Dockerfile restructure and the publish-script
helper extraction are both correct and reduce duplication; the ubuntu image
is a real user request (musl breakage with PyTorch ROCm is a known pain
point). Before merge: (1) call out the new `:latest` tag explicitly in the
PR body, (2) add `${version}-ubuntu` / `${Script.channel}-ubuntu` tags so
ubuntu users can pin, (3) add `&& rm -rf /tmp/dist` to the `RUN` block, and
(4) document supported `BASE_IMAGE` values in a Dockerfile comment.
