# block/goose #8987 — Fix CRT linkage in Windows CUDA build

- **Head SHA:** `0840bb0d6981150000eb99e4576f34bde1f18b9b`
- **Size:** +113 / -67 across 5 files
- **Verdict:** **needs-discussion**

## Summary
Resolves a Windows CUDA build issue by upgrading `candle` to 0.10 (which switches
from `bindgen_cuda` to `cudaforge`) and patching `cudaforge` to honor the CRT
linkage from cargo target-features. Removes the `LLAMA_STATIC_CRT=1` environment
variable and the `target-feature=+crt-static` rustflag override from
`build-cli.yml` and `bundle-desktop-windows.yml`, allowing normal dynamic CRT
linkage for CUDA Windows builds.

## Strengths
- Removing the static-CRT override is the correct direction for Windows: forced
  static CRT in mixed-runtime DLL ecosystems (CUDA SDK in particular) is a known
  source of CRT-mismatch heap corruption.
- The workflow simplification (`.github/workflows/build-cli.yml` around line 157
  and `bundle-desktop-windows.yml` around line 112) drops two layers of override
  and just runs `cargo build --release --target x86_64-pc-windows-msvc -p ... --features cuda`,
  matching how the non-CUDA branch already works.
- `Cargo.lock` shows clean removal of `bindgen_cuda` — no orphaned transitive
  dep.

## Concerns
- This depends on a **patched fork of `cudaforge`** that the author says has a PR
  submitted upstream. Until that upstream PR merges, goose carries a fork patch
  in `Cargo.toml`. Worth flagging:
  - Where is the `[patch."crates-io".cudaforge]` block? Confirm the fork URL, the
    rev, and that it is a public repo with a license compatible with goose's MIT
    setup.
  - What is the exit plan if the upstream PR is rejected or stalls? A comment in
    `Cargo.toml` near the patch with a tracking-issue link would make this
    sustainable.
- `windows-sys` downgrade from 0.61.2 to 0.60.2 in `Cargo.lock` (visible at the
  `anstream`/`anstyle-query` deps) is a pull-back. Was that intentional, or a
  transitive consequence of the candle bump? If unintentional, lock-file solver
  could be coaxed back to 0.61.x.
- The candle 0.10 bump itself is a major-feeling change; please call out in the
  PR description any user-visible behavior changes (numerical differences, new
  GPU memory characteristics, different supported compute caps).

## Recommendation
Hold for discussion until the cudaforge fork situation is documented in-repo and
the candle 0.10 bump's behavioral surface is explicitly enumerated. Once those
are addressed and the upstream cudaforge PR has visible movement, this is a
straightforward landing.
