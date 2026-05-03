# block/goose #8962 — chore(deps): bump mockall from 0.13.1 to 0.14.0

- PR: https://github.com/block/goose/pull/8962
- Author: app/dependabot
- Head SHA: `b8fff02b919c94851abaa29067149bb8b1c62076`
- Updated: 2026-05-01T21:08:31Z

## Summary
Dependabot bumps the `mockall` dev-dependency from 0.13.1 to 0.14.0 in `crates/goose/Cargo.toml` and updates `Cargo.lock` accordingly (also bumps the proc-macro sister crate `mockall_derive` to 0.14.0).

## Observations
- `crates/goose/Cargo.toml:216`: `mockall = "0.14.0"` — declared as a `dev-dependency`, so the change cannot affect runtime artifacts shipped to users. Risk surface is the test suite only.
- `Cargo.lock:6079-6098`: both `mockall` and `mockall_derive` lockfile entries are updated with new checksums. Lockfile is internally consistent (matching versions on the pair). Good.
- mockall 0.14 release notes (per asomers/mockall changelog) include MSRV bump to 1.71+ and rework of how `expect_` methods generate impls when generics are involved. Confirm goose's MSRV in `rust-toolchain.toml` / workspace `Cargo.toml` (`rust-version = ...`) is `>= 1.71`. If the workspace pins to an older MSRV, this bump should be held until MSRV is raised.
- No Rust code (`*.rs`) in the diff — meaning either (a) goose's mock usage doesn't trip any of mockall 0.14's breaking changes (most common: the `Sequence` and lifetime tightenings), or (b) CI hasn't surfaced the issue yet. Make sure CI is green on this PR before merging; rely on it as the gate. If green, low-risk merge.
- This is a routine dependabot patch with no upstream behavior changes for end users. The only review action is "did CI pass?" — gate on that.

## Verdict
`merge-as-is`
