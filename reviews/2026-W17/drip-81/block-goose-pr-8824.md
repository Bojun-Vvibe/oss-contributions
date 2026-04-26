# block/goose PR #8824 — chore(deps): bump anstream from 0.6.21 to 1.0.0

- **PR:** https://github.com/block/goose/pull/8824
- **Author:** app/dependabot
- **Head SHA:** `5b0927a5666a`
- **Stats:** +15 / -39 across 2 files (`Cargo.lock` + `crates/goose-cli/Cargo.toml`)
- **Verdict:** `merge-after-nits`

## What this changes

Dependabot bumps the `anstream` direct dependency in `goose-cli` from 0.6.21 → 1.0.0, with the corresponding lockfile collapse. `crates/goose-cli/Cargo.toml:58` flips `anstream = "0.6.18"` to `anstream = "1.0.0"`. The lockfile diff is mostly bookkeeping: the duplicate `anstream 0.6.21` entry and its sibling `anstyle-parse 0.2.7` get deleted (they were only there because `goose-cli` was on the older line), and every reference to `anstyle-parse 1.0.0` loses its disambiguating version suffix.

The anstream 0.x → 1.0 release is mechanical from the consumer side — the public API (`AutoStream`, `Stripped`, `IsTerminal` re-exports, the `print!`/`println!` macros) is stable, and the bump is mostly a "we're calling it stable now" event after years of 0.6.x patch releases. The lockfile changes confirm this: no source files in `goose-cli` need to change, and the only crate that still pulled `anstream 0.6.21` was `goose-cli` itself, so the duplicate-version footprint cleanly disappears.

## What deserves a closer look before merge

**The lockfile shows simultaneous downgrades of `windows-sys` from `0.61.2` to `0.60.2`, `0.59.0`, `0.52.0`, and `0.48.0` across at least eight different transitive dependents** (`anstyle-query`, `anstyle-wincon`, `dirs-sys`, `fastrand`, `ipnet`, `parking_lot_core`, `rustix`, `tempfile`, `winnow`, etc., visible at lockfile lines 57, 65, 84, 93, 111, 120, 129, 138, 147, 156, 165). That's not how a single direct-dep bump should behave — it indicates the resolver is recomputing the whole dependency closure and picking older `windows-sys` versions because the newer transitive constraints were being pulled by *something* in the previous tree that this bump is now removing. The most likely culprit is that `anstream 0.6.21`'s `windows-sys 0.61.2` floor was the only thing forcing the workspace onto the latest line, and dropping it lets Cargo de-duplicate down to whatever the rest of the tree minimally requires.

That's almost certainly *fine* — `windows-sys` 0.48 / 0.52 / 0.59 / 0.60 / 0.61 are all in production use and the unification is net-good for binary size — but it deserves a sanity check that no goose crate is calling into a `windows-sys 0.61`-specific symbol that was being pulled transitively. A `cargo tree -i windows-sys:0.61.2` on the pre-bump branch followed by the same on the post-bump branch should make the reachability story explicit. If nothing in `goose-*` was a direct caller, the bump is safe.

## Other nits

- The `Cargo.toml` line at `crates/goose-cli/Cargo.toml:58` shows the comment-style annotation `anstream = "0.6.18"` → `anstream = "1.0.0"`, but the *previous* lockfile entry was 0.6.21, meaning the manifest had been pinned below the resolved version. That's harmless, but the new pin to a plain `"1.0.0"` will silently float to 1.0.x patches via Cargo's caret-default behavior — confirm that's intended (`= "1.0.0"` for an exact pin, `"1.0.0"` for caret).
- The bump should run goose's full CI matrix (Linux, macOS, Windows) before merge, since the `windows-sys` shake-out reaches into Windows-only console handling that the `goose-cli` test harness doesn't otherwise exercise.
- No CHANGELOG entry — a 1.0.0 dep bump is worth a one-liner in the release notes if goose maintains a user-facing changelog.

## Bottom line

Direct bump is safe in isolation, lockfile cleanup is correct, but the cascade of `windows-sys` downgrades across the workspace deserves a `cargo tree -i windows-sys` audit before merge. Otherwise mechanical and ready.
