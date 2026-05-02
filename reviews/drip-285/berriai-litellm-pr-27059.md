---
repo: BerriAI/litellm
pr: 27059
head_sha: 8b48cb0e59c646ae4830ff6362fa0411aeff814f
title: "feat(azure_ai): add Grok 4.20 model metadata"
verdict: merge-after-nits
reviewed_at: 2026-05-03
---

# Review: BerriAI/litellm#27059 — `add Grok 4.20 model metadata`

**Head SHA:** `8b48cb0e59c646ae4830ff6362fa0411aeff814f`
**Stat:** +192 / −0 across 3 files (two JSON catalogs + one focused test).

## What it adds

Two new entries in both `model_prices_and_context_window.json` and the
`litellm/model_prices_and_context_window_backup.json` mirror, plus a
test file `tests/test_litellm/llms/azure_ai/test_azure_ai_grok_420_metadata.py`.

The two model entries are `azure_ai/grok-4-20-reasoning` and
`azure_ai/grok-4-20-non-reasoning`. Pricing identical for both:

```json
"input_cost_per_token": 2e-06,
"output_cost_per_token": 6e-06,
"max_input_tokens": 256000,
"max_output_tokens": 256000,
"max_tokens": 256000,
```

The reasoning variant additionally sets `"supports_reasoning": true`; both
set `supports_function_calling`, `supports_vision`, `supports_tool_choice`,
`supports_response_schema`, `supports_web_search`, and a
`supported_modalities: ["text", "image"]` / `supported_output_modalities: ["text"]`
pair.

## Assessment

- The catalog edits are inserted alphabetically right next to the
  pre-existing `azure_ai/grok-code-fast-1` entry (around line 6890 in the
  backup catalog), which matches the file's sort convention.
- Both files are kept in lockstep (same delta in both). Easy to break
  this invariant by hand; this PR gets it right.
- Test file is scoped to model-info lookup + cost calculation for these
  two new IDs — exactly the right surface to lock down. No tests were
  modified that touch other models.
- Pricing source is cited via the `"source"` field on each entry, which
  is the project's documented pattern.

## Nits

1. **`max_input_tokens == max_output_tokens == max_tokens == 256000`.**
   This is unusual. For most providers `max_output_tokens` is materially
   smaller than `max_input_tokens` (model can read more than it writes
   in one shot). Worth double-checking against the upstream model card
   that the output cap really is 256k and not, say, 32k or 64k —
   getting this wrong silently caps user generations or, worse, lets
   them request more than the backend will actually deliver and surface
   confusing 400s.

2. **No cache-pricing fields.** Grok 4.x is generally marketed with
   prompt caching. If the upstream announcement lists separate
   `input_cost_per_token_cache_hit` / `cache_creation_input_token_cost`
   numbers, those should land here too — otherwise cost tracking will
   over-bill cached tokens. Non-blocking if upstream simply doesn't
   publish a cache-discount tier yet.

3. **`supports_web_search: true`.** Confirm the Azure AI deployment
   actually exposes the web-search tool surface (some Grok deployments
   do, some don't depending on tier). If it doesn't, the field will
   advertise a capability that returns errors at call time.

## Verdict

**`merge-after-nits`** — net-additive metadata change, no behavior
risk to other models. The token-cap and cache-pricing nits are
worth a maintainer eye before landing because they're easy to get
wrong and hard to walk back later (third-party cost dashboards
snapshot these numbers).
