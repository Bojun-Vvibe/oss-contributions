---
pr: 26504
repo: BerriAI/litellm
sha: 2fc6b30105a851f14031cbec36eb0eded97ef493
verdict: merge-after-nits
date: 2026-04-25
---

# BerriAI/litellm#26504 — feat: add FuturMix as named OpenAI-compatible provider

- **URL**: https://github.com/BerriAI/litellm/pull/26504
- **Author**: Gzhao-redpoint

## Summary

Adds FuturMix.ai as a named entry in
`litellm/llms/openai_like/providers.json` so users can route via
`futurmix/<model>` instead of falling through the generic
`openai_like/` path. Single-file diff, 8 added lines.

## Reviewable points

- The full diff is one JSON object inserted after the last existing
  provider:

  ```json
  "futurmix": {
    "base_url": "https://futurmix.ai/v1",
    "api_key_env": "FUTURMIX_API_KEY",
    "api_base_env": "FUTURMIX_API_BASE",
    "param_mappings": {
      "max_completion_tokens": "max_tokens"
    }
  }
  ```

  This matches the schema used by the immediately-preceding entries
  (each has `base_url`, `api_key_env`, `api_base_env`,
  `param_mappings`). Mechanically correct.

- **`max_completion_tokens` → `max_tokens` mapping**: this is the
  normal compatibility shim for OpenAI-compatible gateways that
  haven't yet adopted the `max_completion_tokens` field. Worth
  confirming with the FuturMix API: if they *do* support
  `max_completion_tokens` natively (as a Claude/GPT/Gemini aggregator
  they may proxy it through), this mapping will silently downgrade
  reasoning-aware routing to the legacy field. Recommend the author
  test against their reasoning models specifically.

- **Trailing-comma JSON validity**: the diff adds a `,` after the
  previous closing `}` and a new top-level entry. Valid JSON. Good.

- **No tests**: this is consistent with how other entries in
  `providers.json` were added (it's a config table, not a code path),
  so not blocking. But the litellm test suite has a pattern of
  parametric "every named provider parses" tests — confirm the new
  entry is picked up by whichever fixture iterates this file
  (commonly via `pytest.mark.parametrize` over the JSON keys). If
  there's a manual provider list elsewhere (e.g. docs index, model
  list), updating that should be a follow-up but doesn't block this
  PR.

- **Docs**: the PR body has user-facing usage examples. Those should
  also land in `docs/my-website/docs/providers/` or whichever
  directory holds the per-provider doc; PR currently doesn't include
  a docs file. Nit.

- **No banned strings, no secrets, no env defaults that ship a key.**
  Clean.

## Rationale

Single-file provider registration. The schema is right, the field
mappings match siblings. Two non-blocking nits:
(1) verify `max_completion_tokens → max_tokens` is the right
    direction for FuturMix's reasoning-capable models, and
(2) add the matching `docs/providers/futurmix.md` (or equivalent) so
    users can find the integration without reading source.

Otherwise mergeable.

## What I learned

When an OpenAI-compatible aggregator is added to a router, the most
common silent regression is the parameter-mapping table downgrading
new fields to old ones (`max_completion_tokens` → `max_tokens`). It's
the right default for "we don't know yet", but every new aggregator
deserves a short compatibility note in the PR confirming whether the
target API natively understands the newer field.
