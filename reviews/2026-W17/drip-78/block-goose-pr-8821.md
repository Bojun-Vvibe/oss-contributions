# block/goose PR #8821 — chore(deps): bump the cargo-minor-and-patch group with 24 updates

- **PR:** https://github.com/block/goose/pull/8821
- **Head SHA:** `4d7406a62a5362a046129315ffaa5b69a91530a7`
- **Files:** 6 (+186 / -141)
- **Verdict:** `merge-after-nits`

## What it does

Dependabot grouped bump across `Cargo.lock`, root `Cargo.toml`, and the four
crate manifests (`crates/goose-cli/Cargo.toml`, `crates/goose-mcp/Cargo.toml`,
`crates/goose-server/Cargo.toml`, `crates/goose/Cargo.toml`) covering 24
minor-and-patch upgrades. From the PR body table, notable bumps include:

- `axum` 0.8.8 → 0.8.9 (HTTP server)
- `clap` 4.6.0 → 4.6.1 (arg parser)
- `tracing-appender` 0.2.4 → 0.2.5
- `uuid` 1.22.0 → 1.23.1 (one minor)
- `webbrowser` 1.2.0 → 1.2.1
- `zip` 8.4.0 → 8.5.1 (one minor — review zip release notes for archive-handling fixes)
- `rayon` 1.11.0 → 1.12.0 (one minor)
- `tree-sitter` (truncated in PR body)

`Cargo.lock` shows the matching SemVer-clean rev bumps; the manifest edits
appear to be `version = "X.Y.Z"` literal pins moving in lockstep.

## Specific reads

- `Cargo.lock:5,17,29,...` — the diff hunks are uniformly the `version = "X.Y.Z"` + `checksum = "..."` 3-line shape characteristic of patch/minor bumps with no transitive-graph reshuffling. No deletions of dependencies, no new transitive deps showing up — clean grouped patch.
- `Cargo.lock:4730-4735` — appears to add new `tree_sitter_c_sharp-0.23.1-cp310-abi3-*` wheel checksums; this looks like a Python wheel addition leaking into a Rust lockfile, which is impossible — confirm this hunk is actually in `enterprise/poetry.lock` and not `Cargo.lock` (the diff header claims `Cargo.lock` at line 1; if `tree-sitter-c-sharp` is consumed via PyO3 bindings or the `tree-sitter` Rust crate's own packaging it'd look different). Worth a 30s sanity check that the diff header is right.
- The grouped bump uses Cargo's default `caret`/`tilde` semantics; all bumps are MINOR or PATCH per Cargo SemVer, so any caret-ranged dep stays compatible.

## Nits before merge

1. **CI green required**: a 24-package grouped bump touching `axum` (HTTP boundary), `zip` (archive parsing — historically a CVE hotspot), and `rayon` (parallelism primitives) should not merge until the full integration test matrix is green. Confirm the workflow ran `cargo test --workspace --all-features` and a release-mode smoke, not just `cargo check`.
2. **`zip` 8.4 → 8.5 changelog scan**: the `zip` crate has had a steady drip of zip-bomb / path-traversal CVEs in the last 18 months. Skim the 8.5.0 + 8.5.1 release notes; if the bump silently picks up a security fix, call it out in the PR title (so downstream consumers patch faster). If it's pure features, no action.
3. **`axum` 0.8.8 → 0.8.9**: patch-only, but axum middleware ordering has bitten this codebase before. Confirm `goose-server`'s startup smoke still serves `/health` and the WebSocket upgrade path still works under load. Likely fine — this is just here for the paranoia tax.
4. **Verify the `tree_sitter_c_sharp` wheel hunk** is actually in `Cargo.lock` and not a stray paste from `poetry.lock` (see Specific reads above). If real, document why the Rust workspace now needs C# tree-sitter (probably a transitive new dep — name it).
5. **Group-bump rollback strategy**: if any one of the 24 packages turns out to be bad in production, rolling back requires either reverting the whole grouped bump or hand-editing the lockfile. Document in the PR description which package is the highest-risk so a focused revert is possible without un-bumping 23 innocent crates.
6. **No CHANGELOG entry needed** for a dependabot grouped bump; consistent with prior `cargo-minor-and-patch group` PRs.

Mechanical, well-scoped grouped bump. Approve once CI is green and the `zip` release notes are confirmed boring.
