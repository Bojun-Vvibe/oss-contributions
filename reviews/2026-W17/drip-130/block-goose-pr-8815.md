# block/goose #8815 — chore: replace lazy_static with std::sync::LazyLock

- URL: https://github.com/block/goose/pull/8815
- Head SHA: `3b712d87...` (per repo PR list)
- Files: 4 (Cargo.lock, crates/goose/Cargo.toml, security/patterns.rs, tests/providers.rs)
- Size: +12 / −17

## Summary

Drops the `lazy_static = "1.5.0"` dependency from `crates/goose` and
migrates the two visible `lazy_static!` macro call-sites to
`std::sync::LazyLock`, which has been stable in std since 1.80 (and
the workspace pins `rust-version = "1.91.1"`, so the migration is
safe to bottom out on std). Net `−17/+12` line delta plus one fewer
external dependency.

## Specific references

- `Cargo.toml:101` removes `lazy_static = "1.5.0"` from
  `crates/goose/Cargo.toml`. `Cargo.lock:4375` correspondingly drops
  the `lazy_static` entry from the `goose` package's deps.
- `security/patterns.rs:1-3` swaps `use lazy_static::lazy_static;` for
  `use std::sync::LazyLock;`. The migration at `:355-363` rewrites the
  `lazy_static! { static ref COMPILED_PATTERNS: HashMap<...> = {...} }`
  block to `static COMPILED_PATTERNS: LazyLock<HashMap<...>> = LazyLock::new(|| {...})`.
  Semantically identical (same one-time-init, same Sync access pattern)
  but more idiomatic — `LazyLock` is the std-blessed shape for this
  use case and reads as a normal static + closure init rather than a
  custom macro DSL.
- `tests/providers.rs:93-95` migrates two more call-sites:
  `TEST_REPORT: Arc<TestReport>` and `ENV_LOCK: Mutex<()>`. Note these
  are *test-only*, so the migration here also implicitly cleans up the
  test-side dep on `lazy_static` (which presumably wasn't declared as
  a separate dev-dep — it was riding on the runtime dep that the PR
  is removing, so removing the runtime dep would have broken tests
  without this hunk).

## Risk

Near-zero. `LazyLock<T>` and `lazy_static!`-generated wrappers have
the same observable semantics: thread-safe one-time initialization,
`Deref<Target=T>` access, no poisoning on init panic in either
(LazyLock will retry on next access if the closure panics, but for
these init closures (HashMap construction from a static slice, an
`Arc::new`, a `Mutex::new(())`) there's no panic surface). The
`fn(...) -> T` shorthand at `tests/providers.rs:94`
(`LazyLock::new(TestReport::new)` — passing the function pointer
directly instead of a closure) is the right cleanup since
`TestReport::new` happens to have the right signature.

## Nits (non-blocking)

- The PR title says "replace lazy_static with std::sync::LazyLock"
  but only migrates *visible* call-sites in two files. A quick
  `grep -r "lazy_static" crates/` would confirm whether other
  crates in the workspace still depend on `lazy_static` — if so,
  the dep removal from the *workspace*-wide `Cargo.toml` (if it
  lives there) wouldn't be safe. The diff shows the removal is
  scoped to `crates/goose/Cargo.toml`, which is the right narrow
  scope, but the PR title implies broader reach than the change.
- No test added that pins `COMPILED_PATTERNS` actually initializes
  on first access — this would be a behavioral regression test for
  a refactor that's claimed to be behavior-preserving. A
  `pattern_matcher_loads_threat_patterns_on_first_use` test that
  calls `PatternMatcher::new()` and asserts non-empty pattern set
  would close coverage.
- `LazyLock::new(|| Mutex::new(()))` for `ENV_LOCK` could use
  `LazyLock::new(Mutex::default)` since `Mutex<()>::default()`
  exists — micro-style, take it or leave it.
- Worth a one-line comment in `security/patterns.rs` documenting
  that `COMPILED_PATTERNS` skips patterns whose `Regex::new` fails
  silently (`if let Ok(regex) = ... { patterns.insert(...) }`) —
  the prior `lazy_static!` form had the same silent-skip behavior
  but the inline closure shape makes the silent-skip more visible
  and worth calling out as deliberate (otherwise a future reader
  may "fix" it by panicking on bad patterns, which would crash the
  whole binary on a typo'd entry in the static `THREAT_PATTERNS`
  slice).

## Verdict

`merge-as-is` — clean dep-reduction refactor, idiomatic std
migration, behavior-preserving by construction. Ship.
