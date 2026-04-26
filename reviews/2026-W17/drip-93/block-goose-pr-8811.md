---
pr: 8811
repo: block/goose
sha: e59c54cb3571ad737f114db8c1ed7349d5a359bd
verdict: merge-as-is
date: 2026-04-27
---

# block/goose #8811 — chore: move insta to dev-dependencies

- **Author**: r0x0d
- **Head SHA**: e59c54cb3571ad737f114db8c1ed7349d5a359bd
- **Link**: https://github.com/block/goose/pull/8811
- **Size**: +1/-1 in `crates/goose/Cargo.toml`.

## Scope

Trivially correct hygiene fix. `insta` (the snapshot-testing crate) was declared at `[dependencies]` in `crates/goose/Cargo.toml`, which puts it in the runtime build graph for every dependent of `goose`. It belongs at `[dev-dependencies]` since it's only used by snapshot tests.

## Specific findings

- `crates/goose/Cargo.toml:166` — removal of `insta = "1.43.2"` from the `[dependencies]` block (between the `schemars` features-list close and `shellexpand`). Fine — `insta`'s public API is `assert_*` macros that only get pulled in by `#[cfg(test)]` code paths, so no `pub use` re-exports could plausibly depend on it.
- `crates/goose/Cargo.toml:228` — addition of `insta = "1.43.2"` to the `[dev-dependencies]` block, alongside `goose-test-support`, `bytes.workspace = true`, `http.workspace = true`, and `goose-mcp` (which is itself a path dep). Version pin preserved exactly, no version bump bundled in.
- The diff is two lines (one `-` plus one `+`) on the same string with the same version, which is the minimum viable change. No `Cargo.lock` churn shown in the diff (likely because moving between `[dependencies]` and `[dev-dependencies]` doesn't change the resolved version).

## Risks

Effectively zero. The only failure mode would be if some non-test code path in `crates/goose` imports `insta::*` outside `#[cfg(test)]`, which would now fail to compile — but that pattern would itself be a bug worth catching.

## Verdict

**merge-as-is** — one-character net change, fixes an obvious dependency-classification mistake, no risk surface. Exactly the kind of janitorial PR that should land without ceremony.
