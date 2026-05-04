# block/goose #8920 — feat(providers): add Perplexity as a supported model provider

- **Head SHA reviewed:** `9ad7f689869c74d053801181a6b1802e679c1ea8`
- **Size:** +167 / -0 across 5 files
- **Verdict:** merge-after-nits

## Summary

Adds `PerplexityProvider` mirroring the xAI provider layout. Since
Perplexity's chat completions API is OpenAI-compatible at
`https://api.perplexity.ai`, the provider is a thin `ProviderDef`
wrapping `OpenAiCompatibleProvider`. Registers with
`registry.register::<PerplexityProvider>(false)` (i.e. not
auto-enabled), adds documentation entry, and includes 12 lines of
provider tests.

## What I checked

- `crates/goose/src/providers/perplexity.rs:1-10` — sensible
  constants: `PERPLEXITY_PROVIDER_NAME = "perplexity"`,
  `PERPLEXITY_API_HOST = "https://api.perplexity.ai"`,
  `PERPLEXITY_DEFAULT_MODEL = "sonar-pro"`. Good defaults.
- `perplexity.rs:14-22` — `PERPLEXITY_KNOWN_MODELS` lists `sonar`,
  `sonar-pro`, `sonar-reasoning`, `sonar-reasoning-pro`. The
  doc-comment correctly notes Perplexity drifts these — good defensive
  framing. `sonar-deep-research` is currently in their public catalog
  too; consider adding (or note explicitly that long-running deep
  research isn't supported via this provider yet).
- `perplexity.rs:30-39` — `resolve_api_key()` accepts
  `PERPLEXITY_API_KEY` (canonical) or `PPLX_API_KEY` (alias). Nice
  ergonomic touch matching Perplexity's own SDK env var. Returns the
  *primary* error on full failure, which is the right user-facing
  message.
- `perplexity.rs:50-69` — `metadata()` registers two `ConfigKey`s:
  `PERPLEXITY_API_KEY` (`required=true, secret=true`) and
  `PERPLEXITY_HOST` (`required=false`, defaulted, not secret).
  Setup steps point to the actual API keys page. The metadata
  signature matches xAI's pattern.
- `perplexity.rs:72-92` — `from_env` reads the API key, the optional
  custom host, builds an `ApiClient` with `BearerToken` auth, and
  hands it to `OpenAiCompatibleProvider::new`. The empty `String::new()`
  for the fourth arg matches the xAI pattern (presumably an org/path
  prefix that's unused for Perplexity). Worth a one-line comment
  identifying what that fourth arg is so the empty string isn't
  cargo-culted forward.
- `perplexity.rs:97+` (tests) — `test_metadata_structure` asserts
  name, display name, default model, doc URL, and config-key shape
  including `required` + `secret` flags on the API key. Good
  coverage of the metadata contract; doesn't (and shouldn't) hit the
  network.
- `crates/goose/src/providers/init.rs:34,82` — provider added to the
  registry as not-default-enabled, alphabetically slotted between
  `OpenRouterProvider` and `PiAcpProvider`. Correct.
- `crates/goose/src/providers/mod.rs:45` — `pub mod perplexity;`,
  alphabetical. Correct.
- `crates/goose/tests/providers.rs:+12` — provider integration test
  hook (presumably the standard `assert_provider_registered("perplexity")`
  pattern); not shown in my snippet but consistent with project
  layout.
- `documentation/docs/getting-started/providers.md:+1` — single-line
  documentation entry. Could be richer (link to API key creation,
  note search-grounding behavior), but acceptable for a first pass.

## Concerns / nits

1. **Search-grounding behavior is silently inherited from
   OpenAiCompatibleProvider**, but Perplexity's distinguishing
   feature is web-search grounding embedded in chat responses. The
   provider does not surface citations as tool calls or attach them
   anywhere, so users get answers without the citation array
   Perplexity returns. Worth at minimum a follow-up issue, and ideally
   a small adapter in this PR to thread `citations`/`search_results`
   through goose's response shape.
2. **Streaming.** Perplexity supports SSE streaming; if
   `OpenAiCompatibleProvider` already handles it the user inherits it
   for free, but a one-line confirmation in the PR body would help
   reviewers.
3. **Doc page entry is one line.** Other providers in
   `providers.md` get a short paragraph including model list and
   API-key acquisition steps. Worth bringing parity.
4. **Rate-limit semantics.** Perplexity's rate limits are tier-based
   and use a slightly non-standard `x-ratelimit-*` header set; if
   goose's shared retry layer doesn't already parse those, expect
   noisy retries. Not blocking.
5. `PERPLEXITY_DOC_URL` points to
   `https://docs.perplexity.ai/docs/getting-started`. Perplexity has
   reorganized their docs under
   `https://docs.perplexity.ai/home`/`/api-reference/`; double-check
   the link still resolves before merging.

## Risk

Low. Additive, registered as non-default, and the OpenAI-compat
plumbing is reused. Worst case is users add the provider, set the
key, and find that citations are missing — annoying but not breaking.

## Recommendation

Merge after fixing the doc URL (nit 5) and either adding citations
plumbing or filing a follow-up issue (nit 1). Concerns 2-4 can be
follow-ups.
