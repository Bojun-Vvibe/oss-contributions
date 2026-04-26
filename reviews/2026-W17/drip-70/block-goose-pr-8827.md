# block/goose#8827 — chore(deps): bump rcgen from 0.13.2 to 0.14.7

- **Repo**: block/goose
- **PR**: [#8827](https://github.com/block/goose/pull/8827)
- **Head SHA**: `a628f52ca22d`
- **Author**: app/dependabot
- **Base**: `main`
- **Size**: +94 / −4, 2 files

## Context

`rcgen` is the self-signed-cert generator goose-server uses for its local
HTTPS endpoint. 0.13 → 0.14 is a *minor* bump (semver-breaking under
cargo's <1.0 rules) and the upstream changelog notes a switch from
`yasna` to `der-parser` for ASN.1 handling, plus a `time` crate
re-export change.

## Change

`crates/goose-server/Cargo.toml` (visible in the file list, not in the
sampled diff window) is the manifest edit. `Cargo.lock` carries the
real story:

1. **Three new transitive deps** added at line ~258: `asn1-rs 0.7.1`,
   `asn1-rs-derive 0.6.0`, `asn1-rs-impl 0.2.0`. These come in via the
   new `der-parser 10.0.0` block at line ~3010.
2. **`der-parser 10.0.0`** is the headline new transitive — replaces
   the previous `yasna` ASN.1 backend in `rcgen` 0.14.
3. The `rcgen` block itself bumps version + checksum + dependency list,
   shedding `yasna` and gaining `asn1-rs` + `time`.

## Strengths

- Single-purpose dependabot bump, scoped to `goose-server` (TLS-bootstrap
  surface), no consumer code changes implied by the rcgen 0.13→0.14
  API delta (the `generate_simple_self_signed` and `RcgenError` /
  `Error` paths are stable across the bump).
- Integrity checksums (`56624a96882b...` for `asn1-rs 0.7.1`,
  `07da5016415d...` for `der-parser 10.0.0`) match the published
  registry SHAs.
- The new `asn1-rs` family is well-maintained (rusticata project,
  same authors as `nom`, used widely in cert-parsing crates) — not a
  shady supply-chain addition.

## Risks / nits

1. **`time` crate dependency surface grows**: `asn1-rs` pulls `time`
   transitively, which goose may already have via other deps but is
   worth a `cargo tree -i time` confirmation. Multiple `time`
   generations co-existing isn't a hard error but is a binary-size
   smell.
2. **`thiserror 2.0.18` requirement**: `asn1-rs 0.7.1` requires
   `thiserror 2.x`. If goose-server still has consumers on `thiserror
   1.x`, both will ship — verify via `cargo tree -d` whether the dup
   is acceptable. (Most workspaces are mid-migration on this; goose
   probably already has both.)
3. **API delta check**: the `rcgen` 0.14 changelog renames
   `RcgenError` → `Error` and changes the `Certificate::serialize_pem`
   signature to return `Result<String, Error>` instead of
   `Result<String, RcgenError>`. The `goose-server` callers aren't
   in the sampled diff — confirm the consumer code still compiles
   (i.e. either CI is green, or the consumer code already used the
   neutral `Error` re-export). If anything broke, this PR would also
   need a code-side patch.
4. **`continue-on-error` is irrelevant** — but worth noting that
   dependabot doesn't run integration tests for goose-server's TLS
   bootstrap code path. A reviewer should manually `cargo test -p
   goose-server` locally before merging since the rcgen path is part
   of server startup.
5. **Sibling open PR `#8826` bumps `rubato 0.16.2 → 2.0.0`** — also a
   major bump in the same dependabot batch. Consider whether the
   reviewer should batch-review the open dependabot PRs together to
   spot common transitive-dep churn (both pull `time`-family changes).

## Verdict

**needs-discussion** — minor (cargo-semver-breaking) bumps to a TLS
crate deserve a "I ran the test suite locally" comment from a human
reviewer before merging. The transitive churn (3 new crates, 1
backend swap) is normal for a backend swap, but the rcgen 0.14 API
rename means the consumer code in `goose-server` needs explicit
verification — not just a green dependabot CI badge.

## What I learned

Dependabot's "just bump the version" PRs hide the real review surface
when the upstream did a backend swap (here: `yasna` → `der-parser`).
The integration-test harness for cert generation is the only thing
that catches the API-rename case, so any rcgen-touching dependabot
PR is a candidate for a "review by human, not auto-merge" label even
when the manifest delta is one line. Rust's `<1.0 minor = breaking`
convention bites worst on crates that pivot their core data model
between minor versions.
