---
pr: block/goose#8920
sha: 9ad7f689869c74d053801181a6b1802e679c1ea8
verdict: merge-after-nits
reviewed_at: 2026-04-30T00:00:00Z
---

# feat(providers): add Perplexity as a supported model provider

URL: https://github.com/block/goose/pull/8920
Files: `crates/goose/src/providers/{init.rs,mod.rs,perplexity.rs}`, `crates/goose/tests/providers.rs`

## Context

Goose lacked a first-class Perplexity provider; users wanting Sonar models
had to point the generic `OpenAiCompatibleProvider` at
`https://api.perplexity.ai` manually with no setup wizard, no curated
model list, and no doc link. This PR adds a dedicated `PerplexityProvider`
that delegates request-handling to `OpenAiCompatibleProvider` (Perplexity
exposes an OpenAI-compatible chat completions endpoint) but provides
canonical metadata, a setup flow, and a key-resolution alias.

## What changed

`perplexity.rs` (new file, 151 lines):

- Constants at lines 8-22: `PERPLEXITY_API_HOST`, `PERPLEXITY_DEFAULT_MODEL =
  "sonar-pro"`, `PERPLEXITY_KNOWN_MODELS = [sonar, sonar-pro,
  sonar-reasoning, sonar-reasoning-pro]`, doc URL.
- `resolve_api_key()` at lines 28-37: tries `PERPLEXITY_API_KEY` first,
  falls back to `PPLX_API_KEY` (the alias Perplexity's own SDKs use).
  The fallback returns the *primary* error on miss, which is the right
  call — users grepping logs for "PERPLEXITY_API_KEY" find it.
- `ProviderDef` impl at lines 41-95: standard metadata + setup-steps,
  builds an `ApiClient` with `BearerToken` auth, wraps it in
  `OpenAiCompatibleProvider` with empty system prompt.
- Tests at lines 97-150: metadata structure, known-models non-empty,
  setup-steps present, default-model in known-models, doc URL points
  at perplexity.ai.

`init.rs:34,82` and `mod.rs:45` register the provider in the registry
with `register::<PerplexityProvider>(false)` — the `false` matches sibling
non-default providers (PiAcp, Snowflake) and means it is available but
not auto-selected.

`tests/providers.rs:23` imports `PERPLEXITY_DEFAULT_MODEL` for the
shared provider integration suite (pattern matches OpenAI/Google/LiteLLM
sibling imports).

## Nits

- `PERPLEXITY_KNOWN_MODELS` is a hand-maintained list and Perplexity
  rotates models aggressively (the `sonar-huge-online` family was
  retired, `sonar-reasoning-pro` was added recently). The doc-comment at
  lines 14-16 acknowledges this and points at `GOOSE_MODEL` overrides,
  which is the right escape hatch — but consider linking the model-list
  page (`https://docs.perplexity.ai/getting-started/models`) explicitly
  in the setup-steps array so wizard users find new models without
  needing a goose release.
- `PERPLEXITY_DOC_URL` points at `/docs/getting-started`. The current
  canonical Perplexity docs URL has dropped the `/docs` prefix
  (`https://docs.perplexity.ai/getting-started/overview`). Worth
  double-checking the link still resolves before merge.
- `OpenAiCompatibleProvider::new(... String::new())` passes an empty
  system prompt. Perplexity's Sonar models behave noticeably differently
  with vs. without a search-grounding directive in the system prompt —
  not a blocker for shipping the provider, but worth a follow-up issue
  to test default behavior on the reasoning-pro variant.

## Verdict

`merge-after-nits` — clean provider scaffold matching the existing
registry pattern, sensible key alias handling, and tests cover the
metadata contract. The model-list staleness and doc-URL precision are
worth a quick check before merge but neither blocks the feature.
