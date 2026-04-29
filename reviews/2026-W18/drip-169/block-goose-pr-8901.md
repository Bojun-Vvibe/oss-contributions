# block/goose#8901 — `build: set LLAMA_STATIC_CRT for Windows CUDA`

- PR: https://github.com/block/goose/pull/8901
- Head SHA: `a28cd2a0b3f1eda0183336a66a06e460b862c2b0`
- Author: jh-block

## Summary of change

Two-line CI build-config fix. In both
`.github/workflows/build-cli.yml` and
`.github/workflows/bundle-desktop-windows.yml`, adds
`LLAMA_STATIC_CRT: ${{ ... == 'cuda' && '1' || '' }}` to the env block
of the Windows build step that already conditionally sets
`CUDA_COMPUTE_CAP: '80'` for CUDA variants. Tells the bundled
`llama.cpp` build to statically link the MSVC C runtime so the
resulting `.exe` doesn't need `vcruntime140.dll` /
`msvcp140.dll` redistributables on the target machine — important
for the CUDA bundle which ships as a self-contained installer.

## Specific observations

- `.github/workflows/build-cli.yml:160` — the conditional
  `${{ matrix.variant == 'cuda' && '1' || '' }}` produces an empty
  string for non-CUDA variants. `LLAMA_STATIC_CRT=""` is treated as
  unset by `llama.cpp`'s CMake (`if(DEFINED ENV{LLAMA_STATIC_CRT})`
  is false for empty), so this should be a no-op for the CPU
  variant. Correct, but worth confirming on the actual `llama.cpp`
  pin in the repo — some forks check string equality to `1`/`ON`
  and an empty `""` value will short-circuit differently.
- `.github/workflows/bundle-desktop-windows.yml:115` — same pattern
  using `inputs.windows_variant` instead of `matrix.variant`.
  Consistent with the rest of that file's variant gating.
- Static-CRT linking has a real binary-size cost (~1-2 MB per dll
  folded in) and changes redistribution-license obligations for
  the MSVCRT pieces — MS permits redistribution under the
  Visual C++ runtime license, but auditing teams sometimes flag
  static MSVCRT linkage explicitly. Worth one line in the PR body
  acknowledging the licence/size trade.
- The non-CUDA build keeps `LLAMA_STATIC_CRT=""` (so dynamic CRT,
  needs the MSVC redistributable installed). That asymmetry is
  intentional — only the CUDA bundle is the self-contained
  installer where this matters — but a comment in the workflow
  would prevent the next contributor from "normalising" the two
  paths and silently breaking the CPU bundle.
- No test/CI change accompanying this. The verification is
  necessarily empirical: build the CUDA artifact, copy to a clean
  Windows VM without VC++ runtimes, run it, confirm it doesn't
  fail with `vcruntime140.dll not found`. Worth noting in the PR
  description that this was performed.

## Verdict

`merge-after-nits` — correct, minimal, and addresses the right
problem (self-contained CUDA bundle on machines without VC++
redist). The asymmetry deserves a comment, and an empirical
confirmation that the resulting `.exe` actually loads on a clean
Windows machine should be in the PR body before merge.
