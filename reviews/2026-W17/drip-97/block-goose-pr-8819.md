# block/goose#8819: chore(deps): bump actions-rust-lang/setup-rust-toolchain from 1.15.4 to 1.16.0

- **Author:** app/dependabot
- **HEAD SHA:** 8618c416
- **Verdict:** merge-as-is

## Summary

Standard dependabot bump of the
`actions-rust-lang/setup-rust-toolchain` GitHub Action from v1.15.4
(commit `150fca88...`) to v1.16.0 (commit `2b1f5e9b...`). One file
touched: `.github/workflows/ci.yml:122`. The bump replaces a
SHA-pinned action reference with the new SHA-pinned action
reference and preserves the `# v1` trailing comment for human
readability — this is exactly the pattern dependabot should produce
and exactly the security posture (full-SHA pinning instead of tag
reference) that a Rust CI workflow should maintain.

The 1.15.4 → 1.16.0 changelog (transcribed from the PR body) is
narrow and low-risk: a single feature addition (`cache-save-if`
input that propagates to `Swatinem/rust-cache` as `save-if`) plus
a README documentation tweak. There are no breaking changes, no
input renames, no behavior changes for existing workflows, and the
project's `ci.yml:124` invocation only passes `toolchain:` — none
of the inputs touched by this release are in use here. Compatibility
score from dependabot is included in the PR body.

## Specific feedback

- `.github/workflows/ci.yml:122` — the only line changed:
  ```yaml
  -      - uses: actions-rust-lang/setup-rust-toolchain@150fca883cd4034361b621bd4e6a9d34e5143606  # v1
  +      - uses: actions-rust-lang/setup-rust-toolchain@2b1f5e9b395427c92ee4e3331786ca3c37afe2d7  # v1
  ```
  The new SHA `2b1f5e9b...` matches the v1.16.0 release tag from
  the upstream (verified via the PR body's commit list pointing at
  the `Update CHANGELOG for version 1.16.0` commit at exactly that
  SHA). Pin is correct.
- The workflow only passes `toolchain: ${{ steps.msrv.outputs.msrv }}`
  to the action (`ci.yml:124`) — none of the new inputs from this
  release (`cache-save-if`) or from recent prior releases
  (`cache-all-crates`, `cache-workspace-crates`, `cache-provider`,
  `rust-src-dir`) are in use, so the upgrade is purely passive
  for this caller.
- The full commit list in the PR body shows only 5 commits between
  v1.15.4 and v1.16.0, and they are: changelog update, merge of
  PR #90, the `cache-save-if` feature commit itself, merge of PR
  #88, and a README update. No source-of-truth surprises in the
  change set.
- Dependabot compatibility badge included in the PR body — no
  conflicts reported.

## Risks / questions

- `actions-rust-lang/setup-rust-toolchain` is widely used in the
  Rust ecosystem and the v1.x line has a stable input contract,
  so the marginal risk of a behavior regression in this bump is
  near zero. The biggest possible failure would be a transient
  outage in the Rust component download path on the runner —
  unrelated to this PR.
- One non-blocking observation: this is the only `setup-rust-toolchain`
  invocation visible in the diff context, but `ci.yml` is the
  workflow file most likely to have *multiple* uses of this action
  (e.g., for stable + nightly + MSRV). Verify via a repo-wide grep
  whether any other invocations were missed by dependabot — if
  so, dependabot should open a follow-up PR for those, not block
  this one.
- Action SHA pinning is correctly preserved — do not relax to a
  tag reference (`@v1.16.0` or `@v1`) in any future cleanup, as
  full-SHA is a meaningful supply-chain hardening for actions
  that run with repo-write secrets.
- The release was published 2026-04-13 per the changelog, giving
  about two weeks of bake time in the upstream ecosystem before
  this PR — appropriate latency for a low-risk dev-tooling action
  bump.
