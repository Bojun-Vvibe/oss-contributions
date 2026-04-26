---
pr: 2686
repo: charmbracelet/crush
sha: 263e32b6f5be71865cc9da4035b17919fcc90cab
verdict: request-changes
date: 2026-04-27
---

# charmbracelet/crush #2686 — feat: support Moonshot and Moonshot China API keys in config

- **Author**: (community contributor)
- **Head SHA**: 263e32b6f5be71865cc9da4035b17919fcc90cab
- **Size**: +130/-2 across `README.md`, `go.mod`, `go.sum`, `internal/config/load.go`, `internal/config/load_test.go`.

## Scope

Adds env-var-driven API key resolution for the two Catwalk Moonshot provider IDs:
- `InferenceProviderMoonshot` ← `MOONSHOT_API_KEY` (with `KIMI_API_KEY` alias) — international platform.moonshot.ai.
- `InferenceProviderMoonshotChina` ← `MOONSHOT_CN_API_KEY` (with `KIMI_CN_API_KEY` alias) — platform.moonshot.cn.
README documents both vars + the alias relationship; `load_test.go` adds `TestConfig_configureProviders_Moonshot*` and `TestConfig_configureProviders_MoonshotCNAPIKeyFromEnv`.

The PR ships a **`replace` directive in `go.mod`** pointing `charm.land/catwalk` at a fork (`github.com/ne275/catwalk v0.0.0-20260423090449-7d8389d0fddc`) because the upstream Catwalk does not yet expose `InferenceProviderMoonshotChina`.

## Specific findings

- `internal/config/load.go:312-326` — Moonshot international branch: `cmp.Or(env.Get("MOONSHOT_API_KEY"), env.Get("KIMI_API_KEY"))` is the right precedence (vendor-canonical first, alias second). Falls through to `resolver.ResolveValue(p.APIKey)` for config-file resolution if env is empty. Symmetric with existing provider branches (Cerebras, OpenRouter etc.). Looks correct.
- `internal/config/load.go:327-341` — Moonshot China branch: same shape, vendor-canonical `MOONSHOT_CN_API_KEY` first, alias `KIMI_CN_API_KEY` second. Skip-with-warn when `configExists && empty` matches the established pattern.
- `internal/config/load_test.go` (95 new lines) — covers env-var priority, alias fallback, and the China variant. Reasonable coverage.
- `README.md:194-195, 211` — documents both env vars in the table and adds a clarifying paragraph about the `KIMI_*_KEY` aliases. Wording is clear.
- **`go.mod` and `go.sum` — the `replace` directive is the showstopper.** `replace charm.land/catwalk => github.com/ne275/catwalk v0.0.0-20260423090449-7d8389d0fddc` ships crush against an unaffiliated GitHub fork from a personal account. This means:
  - Every crush user pulls a third-party fork of the provider catalog with no upstream supply-chain audit trail.
  - If `ne275` deletes or rewrites the fork, future crush builds break.
  - charm.land's own Catwalk version pin is now decoupled from what gets compiled in.
  - The PR author notes this is "temporary" pending an upstream Catwalk release that adds `InferenceProviderMoonshotChina` — but "temporary" `replace` directives in `main` have a way of becoming permanent.
- The right shape: (1) get the `InferenceProviderMoonshotChina` constant into upstream Catwalk first, (2) bump the `charm.land/catwalk` version in `go.mod` to a release that contains it, (3) *then* land this PR without the `replace`. If maintainers want to land in two phases, the China branch should be guarded behind a build tag or commented out until the upstream pin is bumped.
- The "Moonshot international" half (which only needs the *existing* `InferenceProviderMoonshot` constant that's presumably already in the pinned Catwalk) could be split into its own PR and merged immediately with no `replace` directive.

## Risk

High on the supply-chain axis. Functionality risk is low — the code itself is well-shaped and tested, and the worst-case behavior on a misconfigured user is "skip Moonshot provider with a warning". The risk is entirely in the dependency-pinning story.

## Verdict

**request-changes** — (1) split into two PRs: "Moonshot international env-var support" (no `replace` needed) ships now; "Moonshot China env-var support" lands after upstream Catwalk releases the `InferenceProviderMoonshotChina` constant. (2) Drop the `replace github.com/ne275/catwalk` line from `go.mod` entirely before any merge to `main`. (3) If urgent, ask the Catwalk maintainers (same org, charm.land) for an expedited tag containing the China constant — that's the proper unblock path, not a personal-account fork pin.
