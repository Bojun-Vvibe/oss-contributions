# crush PR #2686 — feat: support Moonshot and Moonshot China API keys in config

Link: https://github.com/charmbracelet/crush/pull/2686

## What it changes

Adds two API-key resolution paths in `internal/config/load.go`:

- `InferenceProviderMoonshot` reads `MOONSHOT_API_KEY` or
  `KIMI_API_KEY`,
- `InferenceProviderMoonshotChina` reads `MOONSHOT_CN_API_KEY` or
  `KIMI_CN_API_KEY`.

Updates the README with the env-var matrix and adds three table-
driven tests under `TestConfig_configureProviders_Moonshot*`.

Crucially, ships with a `replace github.com/ne275/catwalk` directive
in `go.mod` pointing at a fork commit, because the matching
`charm.land/catwalk` provider IDs have not yet been released. PR
description acknowledges this and flags the merge order:

1. Land + release upstream `catwalk` with `moonshot` / `moonshot-cn`.
2. Bump in crush, drop the `replace`.

130 added / 2 deleted.

## Critique

The configuration code itself is fine — pattern-matches existing
provider entries, dual env vars (vendor + brand alias) is the
established shape. The interesting issue is the *delivery* mechanism.

1. **Shipping `replace` to a fork is a release-blocking smell.** A
   `replace` directive in `go.mod` pointing at a personal-account
   commit means: (a) the fork can be force-pushed or deleted out
   from under crush at any time, breaking historic builds; (b) any
   downstream module that imports crush inherits the `replace`,
   which is hostile to library consumers; (c) reproducible builds
   depend on a non-cryptographic ref (commit SHA is fine, but the
   *path* `github.com/ne275/catwalk` is not under maintainer
   control). The right shape is to block this PR on the catwalk
   release, not to land it with the temp-replace and a TODO.
2. **Env-var precedence.** When *both* `MOONSHOT_API_KEY` and
   `KIMI_API_KEY` are set, which wins? The PR doesn't say. The
   tests appear to cover them individually, not jointly. Pick one
   precedence (recommend: vendor-canonical `MOONSHOT_API_KEY`
   wins, brand alias `KIMI_API_KEY` is fallback) and add a test.
3. **`moonshot-cn` provider routing.** The PR adds the key
   resolution but does not (visible in this diff) exercise that
   the China endpoint is actually different from the global one.
   A test that asserts the resolved client config points at the
   `.cn` base URL would catch a future regression where someone
   collapses both providers into one.

## Suggestions

- Hold this PR until `charm.land/catwalk` is published with the
  two providers; merge as a clean `go get` bump, no `replace`.
- Document and test env-var precedence when both vendor + brand
  aliases are set.
- Add a test that asserts the `moonshot-cn` provider resolves a
  China-region base URL distinct from `moonshot`.

Verdict: feature is fine, delivery shape is wrong. Re-submit
without the fork `replace`.
