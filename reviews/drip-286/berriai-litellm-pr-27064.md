---
repo: BerriAI/litellm
pr: 27064
head_sha: 4a919d0ce8fa1ea2035deeb58643d5460736f013
title: "feat(cost): add pricing for azure_ai/gpt-image-2 and gpt-image-to-image"
verdict: merge-after-nits
reviewed_at: 2026-05-03
---

# Review: BerriAI/litellm#27064 — `feat(cost): add pricing for azure_ai/gpt-image-2 and gpt-image-to-image`

**Head SHA:** `4a919d0ce8fa1ea2035deeb58643d5460736f013`
**Stat:** +95 / −0 across 3 files. Author: Dotify71. Closes #26765.

## What it changes

Adds two new entries to both the live registry (`model_prices_and_context_window.json`) and the backup (`litellm/model_prices_and_context_window_backup.json`):

- `azure_ai/gpt-image-2` (live registry lines 2103+, backup lines 2089+)
- `azure_ai/gpt-image-to-image` (alias requested in #26765)

Both entries have identical pricing:

```json
"input_cost_per_token": 5e-06,
"input_cost_per_image_token": 8e-06,
"output_cost_per_token": 1e-05,
"output_cost_per_image_token": 3e-05,
"cache_read_input_image_token_cost": 2e-06,
"cache_read_input_token_cost": 1.25e-06,
"litellm_provider": "azure_ai",
"mode": "image_generation",
"supported_endpoints": ["/v1/images/generations", "/v1/images/edits"],
"supports_vision": true,
"supports_pdf_input": true
```

Adds `test_azure_ai_gpt_image_models_in_cost_map` at
`tests/test_litellm/test_utils.py:3986–4016` asserting both entries
exist in the live registry with the documented cost fields.

## Assessment

- This is a metadata-only PR. Risk is bounded to "wrong pricing
  numbers" and "wrong supported_endpoints / mode" — both visible only
  to users who actually call these models.
- Updating *both* the live registry and the backup JSON is correct
  practice (the backup is what the package ships at install time;
  there have been past bugs where contributors only updated one).
- The new test reads from the live registry only (line 3998:
  `parents[2] / "model_prices_and_context_window.json"`). It does not
  also assert the backup, which is the file users actually load via
  `litellm` at runtime. A second assertion against the backup would
  catch the next "PR only updates one of the two files" regression.
- PR description's table says "Cached Image Input: $2.00" per 1M
  tokens, which corresponds to `cache_read_input_image_token_cost:
  2e-06` (= $2/M). That matches. The PR description does not mention
  `cache_read_input_token_cost: 1.25e-06` (= $1.25/M for cached text
  input) — that's an extra field with no documented source. Should be
  spot-checked against vendor docs.
- "Alias" framing for `azure_ai/gpt-image-to-image` is misleading —
  these are two independent registry entries with the same numbers, not
  an alias. If the intent is genuine aliasing (so a future price change
  to `gpt-image-2` automatically covers `gpt-image-to-image`), that
  would need actual alias support in the cost map. As-is, they will
  drift the next time someone updates one and forgets the other.
- Both entries set `supports_pdf_input: true`. Verify against vendor
  docs — image-generation models accepting PDF input is unusual and
  worth confirming this isn't a copy-paste from a different model
  template.

## Nits

- (Non-blocking) Add an analogous assertion against
  `litellm/model_prices_and_context_window_backup.json` so future
  updates that touch only one file fail CI.
- (Non-blocking) Document the source of the `cache_read_input_token_cost`
  number in the PR description (vendor pricing page link).
- (Non-blocking) If `gpt-image-to-image` is genuinely meant to be the
  same model under a different invocation name, consider adding a
  comment in the JSON entry pointing to `gpt-image-2` so the next
  contributor knows to update both.

## Verdict

**`merge-after-nits`** — straightforward metadata addition with a
proper unit test; the open questions are about completeness of the
test (backup file) and verification of the `supports_pdf_input` flag,
not correctness of the change itself.
