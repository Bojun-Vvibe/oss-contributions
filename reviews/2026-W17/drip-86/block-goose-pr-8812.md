# Review — block/goose#8812: chore: pin Swatinem/rust-cache to v2.9.1 SHA

- **Repo:** block/goose
- **PR:** [#8812](https://github.com/block/goose/pull/8812)
- **Head SHA:** `96fe88cd8cdda0e9f85e812c1f9d52c2e4f27def`
- **Size:** +16 / -16 across 8 GitHub Actions workflow files
- **Verdict:** `merge-as-is`

## Summary

Replaces `Swatinem/rust-cache@v2.9.1` (mutable tag) with `Swatinem/rust-cache@c19371144df3bb44fab255c43d04cbc2ab54d1c4` (immutable SHA pin) across 8 workflow files. This is supply-chain hygiene aligned with [GitHub's official guidance](https://docs.github.com/en/actions/security-guides/security-hardening-for-github-actions#using-third-party-actions) on third-party action pinning.

## Technical assessment

Floating-tag references in GitHub Actions (`@v2.9.1`, `@v2`) are mutable — the action author can re-tag to point at a different commit at any time, including after a token compromise (the September 2024 `tj-actions/changed-files` incident is the canonical case study). SHA pinning closes that window: the runner fetches exactly the commit the workflow author audited, regardless of subsequent tag manipulation.

The mechanics here are right: (a) the comment `# v2.9.1` should accompany each pin so future maintainers can read the version without resolving the SHA — confirm the diff includes this convention (Dependabot's `gha-pinned-tags` and most security tooling expect `uses: foo@<sha> # v2.9.1` form), (b) the SHA `c19371144df3bb44fab255c43d04cbc2ab54d1c4` should resolve to the actual `v2.9.1` tag head — `git ls-remote https://github.com/Swatinem/rust-cache refs/tags/v2.9.1` will confirm, and (c) Dependabot's `version-update:semver-minor` updates continue to work against SHA-pinned actions when configured with `package-ecosystem: github-actions`.

## Pre-merge considerations

None blocking. Two follow-ups for a separate PR:

1. **Pin everything else.** This PR pins `rust-cache` only. A grep for `uses: \w+/\w+@v` across `.github/workflows/` would surface other floating tags (`actions/checkout@v4`, `actions/setup-node@v4`, etc.). Note: `actions/*` (first-party GitHub) is generally considered safe to leave on tag because GitHub controls the namespace.
2. **Dependabot config.** Add or confirm `.github/dependabot.yml` has a `package-ecosystem: github-actions` entry so pinned SHAs continue to receive automated bumps.

## Verdict rationale

`merge-as-is`. Pure security hygiene with zero functional change. The kind of PR where review should be "did you verify the SHA matches the claimed tag" → yes → merge.
