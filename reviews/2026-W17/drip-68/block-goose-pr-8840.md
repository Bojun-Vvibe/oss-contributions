# block/goose #8840 — Add FuturMix provider

- **Author:** FuturMix (vendor)
- **Head SHA:** `b38080bd1fc4110a17f9c98a08ac55a248d4184c`
- **Base:** main
- **Size:** +76 / -0 (2 files)
- **URL:** https://github.com/block/goose/pull/8840

## Summary

Vendor-submitted PR adding `FuturMix` as a new declarative provider.
FuturMix is described as a "unified AI gateway" reselling models from
Anthropic, Google, OpenAI, and DeepSeek through an OpenAI-compatible
endpoint. Pure additive change: one new JSON config file plus a docs
section. Follows the existing declarative-provider pattern (groq.json,
novita.json).

## Specific findings

- `crates/goose/src/providers/declarative/futurmix.json:1-36` (new
  file). Required fields are present and well-formed:
    - `name: "futurmix"` — lowercase, matches file basename.
    - `engine: "openai"` — declares OpenAI-compatible request shape.
    - `api_key_env: "FUTURMIX_API_KEY"` — standard `<PROVIDER>_API_KEY`
      naming convention.
    - `base_url: "https://futurmix.ai/v1/chat/completions"` — **this
      is wrong shape if the OpenAI engine in this codebase appends
      `/chat/completions` itself.** Worth grepping `engine: "openai"`
      consumers to confirm whether `base_url` should be the API root
      (`https://futurmix.ai/v1`) or the full endpoint
      (`https://futurmix.ai/v1/chat/completions`). Existing
      declarative providers (groq, novita) typically configure the
      *root* and let the engine append the path; if FuturMix follows
      the same pattern, this file will produce 404s on
      `/v1/chat/completions/chat/completions`. **High-priority check
      before merge.**
    - `supports_streaming: true` — matches FuturMix's published API
      docs.
- `crates/goose/src/providers/declarative/futurmix.json:8-34` — five
  model entries (claude-sonnet-4, gpt-4o, gemini-2.5-pro,
  deepseek-chat, claude-haiku-4) with `context_limit` and `max_tokens`
  per model. The numbers look right for the named upstream models
  (200K Anthropic, 128K GPT-4o, 1M Gemini 2.5 Pro, 131K DeepSeek), but
  worth a sanity check that FuturMix isn't capping any of these
  smaller than the upstream limit (gateways often enforce stricter
  caps than their underlying providers).
- `crates/goose/src/providers/declarative/futurmix.json:9` —
  `"name": "claude-sonnet-4-20250514"` and line 30 `"name":
  "claude-haiku-4-20250514"`. Pinning to dated model IDs is fine for
  reproducibility but means this list will silently go stale; some
  other providers in this directory use `latest` aliases. Pick one
  convention (probably "dated, owned by vendor") and document it
  somewhere.
- `documentation/docs/getting-started/providers.md:33` — table row
  added in alphabetical position between "Docker Model Runner" and
  "Gemini," consistent with surrounding entries. Cell content matches
  the PR body description.
- `providers.md:698-735` — full setup section with Desktop and CLI
  tabs, paralleling the existing "Groq"/"Novita AI" sections directly
  above. Format is consistent.
- `providers.md:705` — links to
  `https://github.com/aaif-goose/goose/blob/main/.../futurmix.json`.
  **The `aaif-goose` org in this URL doesn't match `block/goose` (the
  canonical repo).** Either the doc was written against the author's
  fork and not updated, or there's a vendor-controlled mirror.
  Either way: link should point to `block/goose` for the upstreamed
  doc.
- `providers.md:702` — the model bullet list duplicates the contents
  of `futurmix.json`. Two sources of truth for the same fact set —
  fine for docs but means a model-list update has to land in two
  places.
- **No tests added.** Declarative providers in this codebase are
  configuration-only and typically don't get per-provider unit tests
  (the engine layer is what's tested). That's defensible.
- **Vendor-submitted PR.** Worth confirming the PR author has agreed
  to the project's contributor terms and that the maintainers are
  comfortable with vendor self-listing (other declarative providers
  appear to follow this pattern, but it's worth flagging policy-wise).

## Verdict

`request-changes`

## Reasoning

The PR is the right shape (additive, follows the established
declarative-provider pattern, includes docs) but ships with two
concrete problems that need fixing before merge:

1. **`base_url` may be wrong.** Comparing against the convention used
   by sibling provider configs (groq, novita) — most likely the
   `base_url` should be `https://futurmix.ai/v1` and the `openai`
   engine appends `/chat/completions` itself. As written, this config
   will probably 404 every request. Trivial one-character fix once
   confirmed.

2. **Documentation link points to the wrong fork** (`aaif-goose/goose`
   instead of `block/goose`). Will produce a broken link as soon as
   the PR is merged into the canonical repo.

Both are mechanical fixes; once corrected (and after a smoke test
confirming the engine actually reaches FuturMix and gets a non-404
response), this is mergeable. The model-list pinning convention and
context-limit sanity check are softer follow-ups.
