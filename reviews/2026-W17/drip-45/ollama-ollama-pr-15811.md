# ollama/ollama #15811 — docs: add RDNA4 / gfx1201 ROCm build instructions to development.md

- **PR:** https://github.com/ollama/ollama/pull/15811
- **Head SHA:** `e7cb21fdb8572538cb059e1af797773d544dfff8`
- **Files changed:** 1 — `docs/development.md` (+36 lines)

## Summary

Adds a new "Building with ROCm for RDNA4 (gfx1200 / gfx1201)" section explaining that the official 0.20.6 binary doesn't ship a ROCm runner, that the source already lists gfx1200/1201 in `AMDGPU_TARGETS` of the `ROCm 7` CMake preset, and gives a copy-pasteable Ubuntu 24.04 build recipe + a verification command.

## Line-level call-outs

- `docs/development.md:128-129` — the framing is correct: pointing at PR #10676 as the tracker for the official build pipeline update is helpful context. Verify that PR # is actually the right tracker (not a stale link); this is the kind of cross-reference that rots after a merge.
- `:130` — "ROCm ≥ 6.3 is sufficient for gfx1201; ROCm ≥ 7.0 is recommended for hipBLASLt-tuned kernels." Good — splits the floor and the recommended ceiling. Consider whether 6.3 is actually verified for gfx1200 too (the section title covers both); if not, narrow the wording.
- `:138-141` — the build snippet uses `cmake --preset 'ROCm 7'` with `-DAMDGPU_TARGETS=gfx1201`. Two notes:
  - The preset name is single-quoted with a space, which works in bash/zsh but trips some readers in copy-paste from the rendered MD if Mintlify/Hugo wraps. Worth a sanity check after rendering.
  - `-DCMAKE_PREFIX_PATH=/opt/rocm` is hard-coded; on systems where ROCm lands at `/opt/rocm-7.x`, this silently fails to discover. A one-liner saying "adjust if your ROCm install lives at a versioned path" would save people an hour.
- `:142` — `go build .` after the `cmake --build` is correct ordering, but readers coming from the MLX section above may miss that the `cmake --install build --prefix dist` line is what produces the binary the verification step uses. The flow `cmake → go build → cmake --install` reads slightly out of order; reordering to `cmake → cmake --install → go build .` (or merging the two cmake calls with comment) would scan more cleanly. Minor.
- `:148-150` — verification grep `inference compute` is good; the expected substring `library=ROCm compute=gfx1201` is exactly what the runtime emits (matches the existing log format). 
- `:152` — link to issue #14927 for benchmarks/transcript is great. Confirm the issue is still open and useful, not closed-as-stale.
- The new section is inserted *after* "go run . serve" but *before* "MLX Engine (Optional)". Placement is fine, but consider whether this belongs as a subsection under an existing "Building with ROCm" header rather than a top-level addition — if no such header exists today, this is the right spot.

## Verdict

**merge-after-nits**

## Rationale

This is exactly the kind of doc PR maintainers love — concrete, tested, links to the tracking issue and benchmarks. Three small polish items: (1) double-check the PR/issue cross-references aren't stale, (2) note that `/opt/rocm` may be versioned on some installs, (3) consider reordering `cmake --install` before `go build .` for clearer flow. None are blocking; a maintainer could fix any of them in a follow-up commit.
