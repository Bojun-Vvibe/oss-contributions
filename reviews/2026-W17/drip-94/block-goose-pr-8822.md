# block/goose #8822 — chore(deps): bump candle-nn from 0.9.2 to 0.10.2

- Author: dependabot (bot)
- Head SHA: `68b0bb14d6b073b939def125422918b6225059c5`
- Files: `crates/goose/Cargo.toml` (+1 / −1), `Cargo.lock` (+165 / −11)

## Specifics

- `Cargo.toml` change is the canonical one-line `candle-nn = "0.9.2"` → `candle-nn = "0.10.2"` bump in the goose crate. Minor-version bump in 0.x means semver-incompatible by Cargo rules (treated as breaking), and the lockfile churn confirms it: `Cargo.lock` ends up with **both** `candle-core 0.9.2` and `candle-core 0.10.2` resolved side-by-side (`Cargo.lock:1588-1640`), same for `candle-kernels` (0.9.2 + 0.10.2 at `:1619-1657`) and `candle-metal-kernels` (0.9.2 + 0.10.2 at `:1671-1690+`). That means *some other dependent in the graph still pins candle-core 0.9.2* and the build is now compiling two parallel candle stacks.
- New transitive `cudaforge` shows up at `Cargo.lock:1657` as a dep of `candle-kernels 0.10.2` (replacing `bindgen_cuda` in 0.9.2). This is a **net-new third-party crate** entering the supply chain — needs provenance check (publisher, license, audit status). Worth a `cargo audit` and a glance at `cudaforge`'s repo before merging.
- `float8` jumps from 0.6.1 → 0.7.0 within the candle 0.10.2 subtree (`Cargo.lock:1623`); `safetensors` from 0.6.x to 0.7.0; `tokenizers` to 0.22.2; `objc2-foundation` / `objc2-metal` newly required for the metal kernels path. None of these are flagged as security advisories at time of writing but `cargo audit` should be run.
- The `candle-metal-kernels 0.10.2` dependency block (`Cargo.lock:1681-1690`) drops `metal` and adds `objc2`/`objc2-foundation`/`objc2-metal` — this is the candle project's migration off the deprecated `metal` crate to the modern `objc2-*` family. Healthy direction but means macOS Metal codepaths in goose are now exercising new bindings.

## Concerns

- **Two candle stacks resolved side-by-side** is a real concern — bloats build time, doubles binary size for the candle codepaths, and risks runtime confusion if any goose code ends up holding tensors from one version and passing them to the other. The fix is to identify the dependent still pinning 0.9.2 (run `cargo tree -i candle-core@0.9.2`) and either bump it or temporarily revert this PR until the upstream catches up.
- Dependabot's PR body almost certainly does not call out the dual-stack situation or `cudaforge` provenance; a maintainer needs to do that 5-minute verification.
- No CI snapshot or benchmark comparison shown. Candle is on the inference hot path, so a quick before/after `cargo bench` (or at least `cargo build --release --timings`) is warranted.

## Verdict

`request-changes` — not because the bump is wrong, but because merging while the lockfile carries two candle major-versions side-by-side is a maintenance hazard. Identify the 0.9.2-pinning dependent, resolve it (or bump that crate too), then re-run. Also `cargo audit` + `cudaforge` provenance check before merge.

