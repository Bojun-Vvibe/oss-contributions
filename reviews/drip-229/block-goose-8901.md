# block/goose #8901 — build: set LLAMA_STATIC_CRT for Windows CUDA

- **PR**: https://github.com/block/goose/pull/8901
- **Head SHA**: `a28cd2a0b3f1`
- **Files reviewed**: `.github/workflows/build-cli.yml`, `.github/workflows/bundle-desktop-windows.yml`
- **Date**: 2026-05-01 (drip-229)

## Context

Build-only fix for the Windows CUDA release pipeline. The CUDA build
of the bundled `llama.cpp` was failing because the build expected the
MSVC C runtime to be statically linked (`/MT` / `/MTd`) for the GPU
codepath; without that, link errors surface late in the release-cut
process. The `LLAMA_STATIC_CRT=1` env var is the upstream-llama-cpp
documented switch that flips its CMake invocation to use the static
CRT. Setting it conditionally per-variant in two workflows is the
minimal fix.

## Diff (2 files, +2 -0)

`.github/workflows/build-cli.yml:160`:

```diff
         env:
           CUDA_COMPUTE_CAP: ${{ matrix.variant == 'cuda' && '80' || '' }}
+          LLAMA_STATIC_CRT: ${{ matrix.variant == 'cuda' && '1' || '' }}
```

`.github/workflows/bundle-desktop-windows.yml:115`:

```diff
         env:
           CUDA_COMPUTE_CAP: ${{ inputs.windows_variant == 'cuda' && '80' || '' }}
+          LLAMA_STATIC_CRT: ${{ inputs.windows_variant == 'cuda' && '1' || '' }}
```

## Observations

1. **Symmetric to the existing `CUDA_COMPUTE_CAP` flag.** The diff
   uses the exact same `${{ matrix.variant == 'cuda' && 'X' || '' }}`
   shape that already gates `CUDA_COMPUTE_CAP=80` for the CUDA
   variant. That's the right precedent to follow — reviewers can scan
   the two-line addition and confirm the conditional matches.

2. **Empty-string default is benign.** When `variant != 'cuda'`,
   `LLAMA_STATIC_CRT=""` is set, which the llama.cpp build treats as
   unset (the CMake check is for non-empty value). No behavior
   change for the CPU / other-variant builds.

3. **Two workflows kept in lockstep.** `build-cli.yml` builds the
   standalone CLI; `bundle-desktop-windows.yml` builds the desktop
   bundle. Both invoke the same llama.cpp build, both need the same
   env var. Updating only one would silently leave the other broken
   on the next release cut. The PR correctly updates both at once.

4. **Conditioned on `matrix.variant` / `inputs.windows_variant`,
   not unconditional.** Setting `LLAMA_STATIC_CRT=1` for non-CUDA
   builds would force static CRT linking on builds where it isn't
   needed and could change the binary's runtime-DLL footprint
   (`vcruntime140.dll` vs static link size delta). Keeping it
   conditional preserves the existing CPU build's CRT-link choice.

## Nits

- **No comment naming the underlying upstream issue.** A one-line
  YAML comment ("# llama.cpp CUDA build requires static MSVC CRT;
  see <upstream issue/PR>") would prevent a future "why is this set
  only for CUDA" cleanup attempt. The PR body links a runs URL but
  not an upstream issue.

- **No release-cut smoke test.** The fix was validated against the
  failing release-cut run cited in the PR body, but there is no
  recurring CI job that exercises the CUDA variant on every PR (the
  release pipelines run on tag, not on PR). A nightly or weekly
  CUDA-variant smoke would catch the next regression of this shape
  before the next release-cut blocks. Out-of-scope for this PR but
  worth a follow-up issue.

- **Two-place env-var setting is begging for an extraction.** The
  pattern of `CUDA_COMPUTE_CAP` + `LLAMA_STATIC_CRT` (and any
  future CUDA-only env vars) duplicated across two workflows is
  the kind of thing that drifts. A composite action or a YAML
  anchor would centralize the CUDA env block. Not blocking for a
  release-unblocking patch.

## Verdict

`merge-as-is` — minimal, surgical, symmetric, and conditional on
the variant that actually needs the change. Nits are documentation
and structural (comment, smoke test, factoring) and shouldn't
block the release-cut unblock.
