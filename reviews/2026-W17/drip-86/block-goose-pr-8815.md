# Review — block/goose#8815: chore: replace lazy_static with std::sync::LazyLock

- **Repo:** block/goose
- **PR:** [#8815](https://github.com/block/goose/pull/8815)
- **Head SHA:** `3b712d87140364b305bbd4802e22e99a5fb177d9`
- **Size:** +12 / -17 across 4 files (`Cargo.toml`, `Cargo.lock`, `crates/goose/src/security/patterns.rs`, `crates/goose/tests/providers.rs`)
- **Verdict:** `merge-as-is`

## Summary

Drops the `lazy_static` crate dependency in favor of `std::sync::LazyLock`, which has been stable in Rust since 1.80 (July 2024). Net −5 lines, removes one transitive dep from `Cargo.lock`. Mechanical replacement at two call sites: a regex pattern map in `crates/goose/src/security/patterns.rs` and a test fixture in `crates/goose/tests/providers.rs`.

## Technical assessment

This is the canonical "modernize idiom" PR. `LazyLock<T>` is the std-library replacement for `lazy_static!` and `once_cell::sync::Lazy` and has identical semantics for the common case (initialize-on-first-deref, thread-safe via `Once`, `&T` accessor via `Deref`). The migration is one-to-one:

```rust
// before
lazy_static! { static ref FOO: Bar = Bar::new(); }
// after
static FOO: LazyLock<Bar> = LazyLock::new(|| Bar::new());
```

Access pattern stays `&*FOO` or `FOO.method()` because `Deref` is preserved. The PR's net −5 line count comes from dropping the `lazy_static!` macro wrapper boilerplate.

Correctness considerations: (a) `LazyLock` requires Rust 1.80+ — confirm the project's MSRV in `rust-toolchain.toml` or workspace `Cargo.toml` allows it; goose has been on recent stable for a while so this is almost certainly fine. (b) Behavior on init panic differs subtly between `lazy_static` and `LazyLock` — both poison on panic, but `LazyLock`'s poison surface returns `LockResult` semantics on subsequent access. For pattern-map and test-fixture initialization that should never panic, this is irrelevant. (c) Removing the `lazy_static` crate from `Cargo.toml` while another workspace member still depends on it would break workspace build — the diff touches `Cargo.lock` so cargo confirmed the removal is safe across the workspace.

## Pre-merge considerations

None blocking. One nice-to-have:

1. **CI matrix MSRV check.** If `rust-toolchain.toml` pins a specific stable channel, this is a no-op. If MSRV is documented separately (some projects pin to a specific minimum), bump it to ≥1.80 in the README/CI config to make the requirement explicit.

## Verdict rationale

`merge-as-is`. Pure modernization, removes a third-party dep in favor of std, no behavior change. The kind of PR that should land same-day.
