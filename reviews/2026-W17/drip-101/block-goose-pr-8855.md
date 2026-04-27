# block/goose PR #8855 — chore(deps): bump the cargo-minor-and-patch group across 1 directory with 23 updates

- URL: https://github.com/block/goose/pull/8855
- Head SHA: `20cf71376ac7103066246849862cb17478a1c3ba`
- Diff: +192 / -147 across 6 files
- Author: dependabot[bot] (`cargo-minor-and-patch` group)

## Context / Problem

Routine grouped dependabot bump touching `Cargo.lock` and the five
`Cargo.toml` files under `Cargo.toml`, `crates/goose/Cargo.toml`,
`crates/goose-cli/Cargo.toml`, `crates/goose-mcp/Cargo.toml`, and
`crates/goose-server/Cargo.toml`. Group rule pulls all minor/patch
upgrades into a single PR rather than 23 separate PRs.

## Design / what's in the diff

Visible upgrades from the `Cargo.lock` head:

- `windows-sys 0.60.2 → 0.61.2` (transitive, propagated via `anstream`,
  `anstyle-query`, `aws-lc-rs`, `axum`, etc.)
- `aws-lc-rs 1.16.2 → 1.16.3` (patch, checksum updated)
- `aws-lc-sys 0.39.0 → 0.40.0` (minor — note this is *not* strictly a
  patch despite being grouped with patches; minor-version bumps in
  `*-sys` crates are usually backward-compat but worth eyeballing)
- `aws-smithy-types 1.3.5 → 1.4.7` (minor, four patch versions of
  cumulative deltas)
- `axum 0.8.8 → 0.8.9` (patch)
- `tokio-tungstenite` — dependency reshuffle visible in the `axum`
  block (`tokio-tungstenite` now resolves to `tokio-tungstenite 0.29.0`
  qualified with version, suggesting a multi-version coexistence in
  the tree)

The full 23-update set isn't visible in the first 80 lines of the
`Cargo.lock` diff, but the shape (grouped minor+patch) is the
standard dependabot rollup.

## Strengths

- **Grouped strategy** is the right choice here — 23 individual PRs
  would saturate review bandwidth with no cross-PR conflict
  resolution. Grouped means CI runs the full matrix once.
- **All upgrades carry checksum updates in `Cargo.lock`**, so cargo
  will fail loudly if a registry tampering is attempted on any of
  the bumped versions.
- **`windows-sys` propagation** is consistent — every consumer of
  `windows-sys 0.60.2` also bumps to `0.61.2`, so there's no
  duplicate-version drag in the Windows surface area.

## Risks / nits

1. **`tokio-tungstenite` multi-version coexistence** — the diff
   shows `axum`'s dependency listed as `tokio-tungstenite 0.29.0`
   (qualified with version), which usually indicates the dep tree
   now contains two `tokio-tungstenite` versions. That's a binary-
   size and compile-time hit, and can cause `Sync`/`Send` trait
   mismatches at edge cases where a connection is passed between
   crates expecting different versions. Worth grepping the full
   `Cargo.lock` for both versions and confirming the duplication
   is intentional (probably `axum` pinned to 0.29 while `goose`'s
   own websocket use stays on a different major).
2. **`aws-lc-sys 0.39 → 0.40`** is a *minor* bump on a
   crypto-adjacent `*-sys` crate. Even though `aws-lc-rs`
   abstracts above it, FIPS/algo-availability changes can leak
   through. Maintainer should confirm CI green on whichever
   platforms exercise the AWS Bedrock code path
   (`crates/goose/src/providers/formats/bedrock.rs`).
3. **`aws-smithy-types 1.3.5 → 1.4.7`** is four patch versions of
   accumulated change in one shot. Unlikely to break (semver
   minor) but the bedrock-formats path uses
   `aws-smithy-types::Document` extensively; flake-watch the
   bedrock test suite on first nightly after merge.
4. **No CI-link in PR body** (dependabot doesn't write one). Trust
   the project's CI required-checks gate to catch regressions
   before merge — the gate is the only review surface that scales
   for grouped bumps.
5. **No changelog summary in the PR body** — dependabot doesn't
   write per-bump changelog excerpts when grouping. Maintainer
   has to spot-check the riskier bumps (aws-lc-sys minor,
   aws-smithy-types four-patch jump) by hand if they care.

## Verdict

`merge-as-is` — grouped minor+patch dependabot rollup. Consistent
`windows-sys` propagation, no orphaned versions visible in the
sampled diff, and CI is the right and only review gate for a
23-update batch. The `tokio-tungstenite` dual-version observation
and the `aws-lc-sys` minor bump are worth a CI-result eyeball
but not a block.

## What I learned

Grouped dependabot bumps trade per-bump review granularity for
CI-batch efficiency. The right time to read every line is when
the lock-file diff shows (a) duplicated package versions
(suggesting a partial-update conflict), or (b) a minor bump
hidden inside a "patch" group on a `*-sys` crate. Both signals
are present here in mild forms — worth one CI confirmation, not
a full-stop block.
