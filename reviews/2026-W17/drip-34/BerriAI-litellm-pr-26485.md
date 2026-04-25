# BerriAI/litellm #26485 — Add FuturMix as named OpenAI-compatible provider

- **Repo**: BerriAI/litellm
- **PR**: [#26485](https://github.com/BerriAI/litellm/pull/26485)
- **Head SHA**: `3cd074622f1c6a2bc652b1389e49f099e7a74b81`
- **Author**: FuturMix
- **State**: OPEN
- **Verdict**: `merge-after-nits`

## Context

A four-key entry added to `litellm/llms/openai_like/providers.json` that
registers `futurmix` as a named OpenAI-compatible provider, with
`base_url=https://futurmix.ai/v1`, env-var keys
`FUTURMIX_API_KEY`/`FUTURMIX_API_BASE`, and a `max_completion_tokens →
max_tokens` param mapping. Vendor onboarding PR; no logic changes
elsewhere.

## Design

The schema at `litellm/llms/openai_like/providers.json:104-110` is
identical in shape to the entries directly above it (the prior provider
already uses the same `max_completion_tokens → max_tokens` mapping at line
~99-103). So the new block is structurally consistent and will flow
through the existing `openai_like` dispatcher without any Python-side
plumbing — that's the whole point of the JSON registry. Sanity check: the
`base_url` resolves to a TLS endpoint, ends in `/v1`, and uses the
canonical OpenAI path layout, which is the bare minimum for the
`openai_like` adapter to work without per-provider overrides.

## Risks / Nits

1. **No `model_prices` rows added.** Most other named providers in this
   registry get at least one entry in
   `model_prices_and_context_window.json` so cost tracking works out of
   the box. Without it, all `futurmix/*` invocations will route through
   the "unknown model" cost branch and log $0 spend silently — this
   matters for users running the proxy with budget enforcement. Either
   add a couple of representative rows or note in the PR description
   that pricing will be a follow-up.
2. **No test fixture.** Sibling provider PRs in this repo typically add
   at least one entry to `tests/llm_translation/test_openai_like.py` or
   the providers-registry test that loads the JSON and asserts the new
   key exists with required fields. A 4-line test would catch a future
   typo in `api_key_env`.
3. **Trailing comma / file shape.** The prior block at line ~103 closes
   with `}` and the new block adds a trailing `}` then file-end. JSON
   parses fine, but contributors editing this file later may want a
   one-line ordering convention (alphabetical? insertion order?) — not
   blocking, just observation.

## What I learned

Vendor-onboarding PRs to a registry-driven adapter look trivial but have
a real downstream contract: budget tracking, retry policy, and rate-limit
classification all key off the provider name. A 9-line PR like this one
is correct in shape but ships with a silent gap (no pricing) that the
operator only discovers after a billing cycle. The right gate isn't
"does the JSON parse" — it's "does adding this provider preserve the
budget-tracking invariants the rest of the registry already provides."
