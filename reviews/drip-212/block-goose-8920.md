# block/goose PR #8920 — feat(providers): add Perplexity as a supported model provider

- Repo: `block/goose`
- PR: https://github.com/block/goose/pull/8920
- Head SHA: `9ad7f689869c74d053801181a6b1802e679c1ea8`
- State: OPEN
- Files: 4 (`crates/goose/src/providers/{init.rs,mod.rs,perplexity.rs}`, `crates/goose/tests/providers.rs`, `documentation/docs/getting-started/providers.md`), +191/−0
- Verdict: **merge-after-nits**

## What it does

Adds Perplexity as a new top-level provider, implemented as a thin
adapter over the existing `OpenAiCompatibleProvider` (since Perplexity
exposes an OpenAI-compatible chat completions endpoint at
`https://api.perplexity.ai`). The provider:

1. **Module wiring** in `crates/goose/src/providers/mod.rs:45` and
   `init.rs:34` + `:83`: declares `pub mod perplexity` and registers
   `PerplexityProvider` with the registry as `enabled=false` (alongside
   `PiAcpProvider`, distinct from `enabled=true` for the well-known
   provider set).
2. **Provider struct** in `crates/goose/src/providers/perplexity.rs`
   (151 lines, all new):
   - Constants: `PERPLEXITY_PROVIDER_NAME = "perplexity"`,
     `PERPLEXITY_API_HOST = "https://api.perplexity.ai"`,
     `PERPLEXITY_DEFAULT_MODEL = "sonar-pro"`,
     `PERPLEXITY_KNOWN_MODELS` = `["sonar", "sonar-pro",
     "sonar-reasoning", "sonar-reasoning-pro"]`,
     `PERPLEXITY_DOC_URL = "https://docs.perplexity.ai/docs/getting-started"`.
   - `resolve_api_key()` (line 67-77) tries `PERPLEXITY_API_KEY`
     first, falls back to `PPLX_API_KEY` (Perplexity's SDK alias),
     and returns the *primary* error if both fail (so the "you
     forgot the canonical key" message is what the user sees).
   - `ProviderDef::metadata()` returns 2 config keys:
     `PERPLEXITY_API_KEY` (required, secret, primary) and
     `PERPLEXITY_HOST` (optional, default = the API host const).
   - `ProviderDef::from_env(...)` reads the API key, reads or defaults
     the host, builds an `ApiClient::new(host,
     AuthMethod::BearerToken(api_key))`, and constructs an
     `OpenAiCompatibleProvider` with empty `String::new()` system
     prompt (i.e. lets the caller's prompt flow through unmodified).
3. **Integration test** in `crates/goose/tests/providers.rs:876-886`:
   gates a real network test behind `PERPLEXITY_API_KEY` env var
   (skipped when absent), targeting `PERPLEXITY_DEFAULT_MODEL`.
4. **5 unit tests** at the bottom of `perplexity.rs`: metadata
   structure, known-models non-empty + contains default, setup-steps
   present + reference the env var, default model is in known set,
   doc URL points to perplexity docs.
5. **Docs** at `documentation/docs/getting-started/providers.md:45`:
   one new row in the providers table with the description "Chat
   models with built-in real-time web search grounding. OpenAI-
   compatible chat completions API at https://api.perplexity.ai" and
   env vars `PERPLEXITY_API_KEY` (or `PPLX_API_KEY`),
   `PERPLEXITY_HOST` (optional).

## What's right

- Right shape: `OpenAiCompatibleProvider` is the existing chokepoint
  for OpenAI-shaped chat APIs; Perplexity's API is one of those, so
  the new file is just the metadata + env wiring shim and contributes
  zero new request/response codepaths. Following the established
  pattern (cf. xAI, OpenRouter, Ollama Cloud) keeps the hot path
  uniform across providers.
- `enabled=false` registration matches the convention for
  third-party providers that need explicit user opt-in (the metadata
  is discoverable but the provider isn't auto-enabled in the picker).
- API-key alias (`PERPLEXITY_API_KEY` primary, `PPLX_API_KEY`
  fallback) is genuinely useful: Perplexity's own Python SDK uses
  `PPLX_API_KEY`, so users who already have that env var set get
  zero-config onboarding. The "return primary error on both-fail"
  policy at `perplexity.rs:73-76` is correct — it surfaces the
  canonical name in the error message rather than the alias, which
  matches what the docs and config UI tell users to set.
- 5 unit tests pin the metadata shape (config-key names + flags +
  default), the known-models list invariant, the setup-steps
  presence, default-in-known consistency, and the doc-URL prefix.
  Right depth for a metadata-only file: tests don't try to mock the
  actual API, they just lock the public contract.
- Empty `String::new()` system prompt on the
  `OpenAiCompatibleProvider` constructor is the right neutral choice —
  Perplexity does not require a provider-specific system prompt to
  function, and adding one would unilaterally inject text into every
  user's conversation.
- Docs row is honest: explicitly mentions the OpenAI-compatible
  endpoint URL, both env var names, and the "real-time web search
  grounding" property that distinguishes Perplexity from a generic
  OpenAI-compatible endpoint.

## Nits (request before merge)

1. **`PERPLEXITY_KNOWN_MODELS` will rot.** The doc-comment at line
   52-54 honestly says "Perplexity ships new and renames existing
   models on its own cadence; this list is a curated default for
   setup wizards. Users can override `GOOSE_MODEL` to point at any
   other model the API accepts." That's correct framing, but the
   list will drift. Consider:
   - Adding a CI link-check or a one-shot script that compares the
     constant against `https://docs.perplexity.ai/...` model list.
   - Or accept the drift and add a `# TODO(perplexity): refresh
     ~quarterly` marker at the constant so the next reader has a
     calendar anchor.
2. **`KNOWN_MODELS.to_vec()` allocates per `metadata()` call.** Line
   90 calls `PERPLEXITY_KNOWN_MODELS.to_vec()` inside `metadata()`,
   which runs at least once per provider registration and possibly
   per UI refresh. Cheap (4 entries), but if `ProviderMetadata`
   accepts `&'static [&'static str]` somewhere, that would be free.
   Pattern matches other providers — not blocking, just noting.
3. **`from_env` ignores `_extensions`.** Line 102-103 takes
   `_extensions: Vec<crate::config::ExtensionConfig>` and ignores it.
   That's likely correct (Perplexity has no extension surface) but
   silent. A one-line comment "Perplexity has no extension hooks;
   the OpenAI-compatible adapter wraps the bare API." would prevent
   the next maintainer from wondering whether they need to thread
   extensions through.
4. **Resolve order of `PERPLEXITY_HOST` and the alias.** The host
   resolves only `PERPLEXITY_HOST`, not e.g. `PPLX_HOST`. If a user
   has `PPLX_API_KEY` set (alias path), they probably also have
   `PPLX_HOST` set. Consider symmetry — either resolve both keys for
   both env vars, or document explicitly that the host is *not*
   aliased.
5. **Integration test placement.** Line 876-886 sits between
   `test_xai_provider` and `test_claude_code_provider` — alphabetical
   order suggests it should come right before the snowflake/together
   block. Trivially fixable, ordering doesn't affect correctness but
   matches the rest of the file's discipline.
6. **`metadata.known_models[i].name`** — the test at line 159 asserts
   `metadata.known_models.iter().any(|m| m.name == ...)`, suggesting
   `known_models` is `Vec<ModelEntry>` (not `Vec<&'static str>`)
   built somewhere in `ProviderMetadata::new`. The doc-string on the
   const says "list" without saying "list of strings" explicitly —
   minor, but worth a one-word "string" qualifier so the next reader
   doesn't have to chase the type.
7. **No CHANGELOG entry** in the diff. Goose appears to track
   user-visible changes in `CHANGELOG.md` or release notes at the
   crate root; a new provider is squarely a user-visible add and
   warrants one line.

## Why merge-after-nits

Clean shim over the existing OpenAI-compatible adapter, all
new-provider invariants satisfied (metadata locked by tests,
registration wired through both `mod.rs` and `init.rs`, docs row
present), and the `PERPLEXITY_API_KEY` / `PPLX_API_KEY` alias is a
real onboarding win. The nits are documentation hygiene (known-models
freshness signal, host-alias symmetry, extensions-unused comment) and
one CHANGELOG entry, none of which block correctness — but landing
them all keeps the provider self-explanatory for the next person who
adds a sibling shim.
