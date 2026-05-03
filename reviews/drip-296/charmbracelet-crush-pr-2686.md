# Review: charmbracelet/crush PR #2686

- **Title:** feat: support Moonshot and Moonshot China API keys in config
- **Author:** ne275
- **Head SHA:** `263e32b6f5be71865cc9da4035b17919fcc90cab`
- **Verdict:** request-changes

## Summary

Adds env-var resolution for Moonshot (Kimi) international and China
endpoints, with `KIMI_API_KEY` / `KIMI_CN_API_KEY` aliases. Config
loader switches on `catwalk.InferenceProviderMoonshot` and
`InferenceProviderMoonshotChina`. README updated. Three new tests cover
the env-key paths.

The functionality is reasonable, but the PR introduces a `replace`
directive in `go.mod` that points to a personal fork of `catwalk`,
which is a non-trivial supply-chain change that should not ship in a
feature PR.

## Specific-line comments

- `go.mod:222` — `replace charm.land/catwalk => github.com/ne275/catwalk v0.0.0-20260423090449-7d8389d0fddc`
  is a hard blocker. A user-fork replace in a published module pulls
  every downstream user onto an unowned, unaudited dependency. The
  upstream `charm.land/catwalk` must add `InferenceProviderMoonshot` /
  `InferenceProviderMoonshotChina` first, then this PR pins the
  released version. Until then the new `case` arms reference symbols
  that only exist on the author's fork.
- `internal/config/load.go:312-326` — the two new `case` arms duplicate
  each other almost exactly; only the env var names and provider label
  differ. Extract a small helper
  `resolveProviderKeyFromEnv(p, prepared, env, resolver, configExists, primary, alias, label)`
  to avoid the copy/paste, especially since the same pattern exists
  for `Hyper` (visible in the surrounding test file context). Reduces
  the surface area for "I added X but forgot Y" bugs.
- `internal/config/load.go:316` and `:331` — `cmp.Or(env.Get("MOONSHOT_API_KEY"), env.Get("KIMI_API_KEY"))`
  resolves the primary first, which is correct; document the
  precedence in the README so users hitting both vars know which wins.
- `internal/config/load_test.go:1593-1683` — three good tests, but no
  test covers (a) the China alias `KIMI_CN_API_KEY`, (b) the case
  where *neither* env var nor config-resolved value is present (should
  delete the provider when `configExists` is true), or (c) the
  precedence assertion when both `MOONSHOT_API_KEY` and `KIMI_API_KEY`
  are set. Worth adding before merge.
- `README.md:194-195` — the two new table rows have inconsistent
  spacing in the Markdown source (one row uses two spaces between
  pipes, the other uses one). Tiny thing, but the rendered table will
  still align — purely a cosmetic nit.

## Risks / nits

- The `replace` directive will get accidentally merged if reviewers
  miss it; this is the one change that *must* be reverted before this
  ships.
- No documentation that `KIMI_*` aliases exist for the China endpoint
  in the env-var table itself (only mentioned in the prose paragraph
  below).

## Verdict justification

The user-facing feature is welcome and the test pattern is good, but
the `go.mod replace` to a personal fork is a hard no for a feature PR.
**request-changes** — drop the replace, wait for upstream catwalk to
expose the constants, dedupe the case arms, and extend the test
coverage.
