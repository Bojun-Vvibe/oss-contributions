---
pr: 8828
repo: block/goose
sha: 4fc099770d5c4b63e55736ce4b7ce3636d2654c5
verdict: merge-after-nits
date: 2026-04-26
---

# block/goose #8828 — chore(deps): bump pctx_code_mode from 0.3.0 to 0.4.0

- **URL**: https://github.com/block/goose/pull/8828
- **Author**: dependabot[bot]
- **Head SHA**: 4fc099770d5c4b63e55736ce4b7ce3636d2654c5
- **Size**: +4/-4 across `Cargo.lock` + `crates/goose/Cargo.toml`

## Scope

Single dependency bump:

- `crates/goose/Cargo.toml:183` — `pctx_code_mode = { version = "^0.3.0", optional = true }` → `^0.4.0`
- `Cargo.lock` — pinned checksum and version flip, plus a transitive bump of `pctx_codegen`'s `itertools` dep from `0.13.0` to `0.14.0` (already present elsewhere in the lockfile, so no new sub-dependency).

`pctx_code_mode` is gated behind an `optional = true` Cargo feature — looking at `crates/goose/Cargo.toml`, it's wired into a non-default feature flag, so 99% of `cargo build`/`cargo test` runs in this repo never even compile the new version.

## Specific findings

- **Major-version bump under SemVer.** `0.3.0 → 0.4.0` on a `0.x` crate is a breaking change by Cargo SemVer convention (every `0.x` minor is a major). The PR body (dependabot's auto-generated text — not visible in the JSON I pulled) should list a CHANGELOG link or release notes. **Nit:** before merging, a maintainer should manually skim the `pctx_code_mode` 0.4.0 release notes to confirm any API the goose tree-sitter integration uses (`pctx_code_mode::*` callsites under the gated feature) didn't get renamed/removed. A `cargo build --features <feature-name>` in CI on the bumped lockfile would catch breakage; if there's no CI matrix entry that exercises the optional feature, this PR is effectively untested by goose's CI.

- **Transitive `itertools 0.13 → 0.14` bump in `pctx_codegen`** is fine — `itertools 0.14` is already in the lockfile (visible from the lock diff at `Cargo.lock:7384` showing the dep being satisfied without adding a new top-level entry). No risk of duplicate transitive trees.

- **`^0.4.0` constraint is correct** under Cargo's caret rules (allows 0.4.x, blocks 0.5.0). Matches the existing `^0.3.0` style.

- **Lockfile-only check.** Because the feature is optional, the standard `cargo check`/`cargo test` will pass without ever touching the new crate version. A maintainer who wants to validate this PR end-to-end should run:
  ```
  cargo check -p goose --features <pctx-code-mode-feature-name>
  cargo test -p goose --features <pctx-code-mode-feature-name>
  ```
  If goose has a `code-mode` or similar feature flag controlling this, the CI pipeline ideally has a job that exercises it. If not, this PR is a "trust dependabot, watch for issues post-merge" call.

- **Author is dependabot[bot].** Standard automated PR. The repo presumably has a routine for these — auto-merge if CI green, manual review if a major bump on a feature-gated crate. This is the latter.

- **No security advisory cited.** Dependabot would have flagged this with a `security` label if there were a CVE on `pctx_code_mode 0.3.x`. The bump is presumably for new features or internal cleanup, not a forced security upgrade. That makes the urgency low.

## Risk

Low-to-medium depending on whether CI exercises the optional feature. If the `pctx_code_mode`-gated build path is only validated on release builds, this lockfile change could land green in PR CI and break the next release. Mitigation: maintainer runs `cargo check -p goose --features <feature>` locally before approving, or adds a CI matrix entry. If that's already the case, risk drops to "almost zero — feature-gated lockfile bump."

## Verdict

**merge-after-nits** — the nit is "confirm CI exercises the optional feature, or run `cargo check --features <feature>` manually." If a maintainer can't verify the feature compiles against 0.4.0, this should be a `request-changes` to add the CI matrix entry first. But assuming the feature gate is in the existing CI, this is a routine green-light.

## What I learned

`0.x → 0.(x+1)` Cargo bumps on **optional/feature-gated** crates are the dependency category most likely to silently break a release. Default-feature CI gives them a green checkmark while skipping all the code that actually uses them. The right hygiene is: every `optional = true` dep gets a CI matrix entry that turns the feature on, even if it just runs `cargo check`. This PR is a useful prompt to verify that's the case for `pctx_code_mode` in goose.
