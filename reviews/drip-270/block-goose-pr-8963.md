# block/goose #8963 — chore(deps): bump sha2 from 0.10.9 to 0.11.0

- URL: https://github.com/block/goose/pull/8963
- Head SHA: `c89de2814994e3e740371979723bcb9ec13de16b`
- Author: @app/dependabot
- Stats: +97 / -42 across 2 files

## Summary

Dependabot bump of the workspace `sha2` dependency in `Cargo.toml`
from `0.10.9` to `0.11.0`, with the corresponding `Cargo.lock` churn.
Because `RustCrypto/hashes 0.11` rests on a new `digest 0.11` /
`crypto-common 0.2` / `block-buffer 0.12` stack (which now depends on
`hybrid-array` instead of `generic-array`), the lockfile carries
both `0.10.x` and `0.11.x` lines of every transitive crate side-by-side
during the migration.

## Specific feedback

- `Cargo.toml:75` — single-line bump of the workspace `sha2 = "0.11"`
  declaration. Any first-party crate in the repo that uses
  `Sha256::new()` / `update()` / `finalize()` against
  `Digest`-trait-bounded code will need to compile against the new
  `digest 0.11` API surface. The trait surface is **not** fully
  source-compatible across the 0.10 → 0.11 jump (`Output`, `OutputSize`,
  `FixedOutput::finalize_into`, and `Reset` semantics moved). PR body
  should confirm `cargo check --all-features` passes — the diff does
  not show source-code adjustments, which is suspicious for a
  workspace this size.
- `Cargo.lock:589`, `:2698`, `:3377`, `:3399` (and ~10 similar) —
  every existing user of `sha2` is pinned to `sha2 0.10.9` via
  transitive deps (`aws-sigv4`, `deno_*`, `ed25519-dalek`, etc.).
  The lockfile correctly carries both `sha2 0.10.9` and `sha2 0.11.x`
  in parallel. This roughly doubles the size of the digest-related
  dependency graph in the binary — measurable but acceptable for a
  CLI tool.
- `Cargo.lock:1369-1375` (`block-buffer 0.12.0`) and `:2399-2405`
  (`crypto-common 0.2.1`) — the new graph picks up `hybrid-array` as
  a fresh transitive dependency. `hybrid-array` is an unstable-ish
  crate that gates const-generics-flavoured array work; worth a quick
  audit of its supply-chain status (publisher, last release cadence)
  before merging.
- `Cargo.lock:2087-2098` — both `const-oid 0.9.6` and `const-oid 0.10.2`
  now resolve. This is fine but slightly increases compile time.
- `Cargo.lock:3104-3118` — both `digest 0.10.7` and `digest 0.11.2`
  resolve simultaneously. This is the most likely source of "two
  versions of `Output<Sha256>` in one binary" gotchas if any code
  passes `Sha256` instances across crate boundaries.
- The PR is purely lockfile + manifest. There is **no** functional
  test demonstrating that the workspace's first-party uses of
  `sha2::Sha256` still compile and behave identically. Strongly
  recommend a follow-up CI job (or proof in the PR description) that
  shows `cargo build -p <each-leaf-crate-using-sha2>` succeeds.
- `sha2 0.11.0` is a **semver-major** bump even though the version
  number is `0.x → 0.y`. Cargo treats `0.10 → 0.11` as breaking. Worth
  flagging in the PR title (Dependabot already does, with `bump`).
- No security advisory associated with `sha2 0.10 → 0.11` at time of
  review; this is purely a feature/refactor release.

## Verdict

`needs-discussion` — the manifest change is one line, but the
0.10 → 0.11 jump in the RustCrypto stack is meaningfully breaking.
Without source-code adjustments in the diff and without a CI run
attached to the PR view, it is not safe to assume the workspace
compiles. Either (a) attach the green CI run, (b) hold this PR until
the workspace's direct `Sha256::*` usages are migrated in the same
PR, or (c) defer until upstream transitives (aws-sigv4, deno_*,
ed25519-dalek) also bump and the duplicate-graph collapses.
