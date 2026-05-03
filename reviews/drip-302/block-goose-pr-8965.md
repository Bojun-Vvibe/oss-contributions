# block/goose PR #8965 — chore(deps): bump nanoid from 0.4.0 to 0.5.0

- Head SHA: `72cedffff6da`
- Author: `dependabot[bot]`
- Files changed:
  - `Cargo.lock`
  - `crates/goose/Cargo.toml` (1 line)

## Analysis

Single-major dependabot bump for `nanoid`. Per the upstream changelog excerpt in the PR body, `0.5.0` brings:

- `rand` dependency bumped to `0.9` (was `0.8`).
- New `rngs::thread_local` random source (#36).
- `format` now accepts any `FnMut(usize) -> Vec<u8>` random generator, enabling seeded and stateful RNGs (#32, #41).

Diff highlights:
- `Cargo.lock:6143-6155` — `nanoid 0.4.0` → `0.5.0`, dependency reflects `rand 0.8.5` → `rand 0.9.2`. New checksum `8628de41fe064cc3f0cf07f3d299ee3e73521adaff72278731d5c8cae3797873`.
- `crates/goose/Cargo.toml:95` — `nanoid = "0.4"` → `"0.5"`.

The interesting transitive effect is the `rand 0.8 → 0.9` dependency. If `rand 0.8` was previously the only consumer of `rand 0.8.x` in the workspace, this bump might let the lockfile drop the old `rand` major entirely. The diff truncates after the `nanoid` block, so the reviewer should confirm in the unshown lock regions whether `rand 0.8.5` is still pulled by other crates (likely yes — `rand` is widely used).

Call-site impact:
- `nanoid::nanoid!()` macro and the basic `nanoid::nanoid!(size)` form did not change between 0.4 and 0.5; the new `format` callback signature only matters for callers that pass a custom RNG.
- A quick `rg "nanoid::" crates/goose/src` would confirm goose only uses the macro form, in which case this is a transparent bump.

Risks:
- The `rand 0.9` axis change is subtle: if any *other* crate in the workspace explicitly depends on `rand 0.8` and shares RNG instances with goose's `nanoid` paths (unlikely), behavior could change. Not visible in this diff.
- No tests added; bumping `nanoid` doesn't require any, but a quick `cargo test -p goose` covering the ID-generating code paths is the cheapest sign-off.

## Verdict

`merge-as-is`

## Nits

- If goose-the-crate also exposes `nanoid` re-exports to downstream consumers, bump the goose crate's MINOR version to flag the transitive `rand` major change in semver.
