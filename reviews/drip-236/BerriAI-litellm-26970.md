# BerriAI/litellm#26970 — feat: add Venice AI models and token prices to providers

- **PR**: https://github.com/BerriAI/litellm/pull/26970
- **Head SHA**: `420e74c9e3cca55460902a5e9522946b6e1a1de7`
- **Size**: +2494 / -184, 2 files
- **Verdict**: **needs-discussion**

## Context

Fixes #24229 — adds Venice AI text models with pricing and context windows to `model_prices_and_context_window.json` (and the backup mirror). Pricing data sourced from Venice AI API `/models?type=text` per PR body, with `$ per 1M tokens` → `input_cost_per_token` conversion applied.

## What's right

**Venice AI model entries** (additions in `model_prices_and_context_window.json`, mirrored in `model_prices_and_context_window_backup.json`) follow the standard JSON shape used by other named providers: `litellm_provider`, `mode`, `max_input_tokens`, `max_output_tokens`, `max_tokens`, `input_cost_per_token`, `output_cost_per_token`, optional `supports_*` capability flags, and `source` URL. Sourcing from Venice's own `/models?type=text` API rather than from a docs page or a screenshot is the right discipline — it makes the data round-trippable and verifiable post-merge.

The PEP-440-style `/1M tokens` conversion is the right unit normalization: LiteLLM's downstream cost math expects per-token costs in scientific notation (e.g. `2.99999e-06`), not the per-million-token figures Venice's API returns. Doing the conversion at ingest time keeps the rest of the cost-calculation pipeline simple.

## Risks / nits — the load-bearing concern

**The visible diff is dominated by spurious churn in unrelated `databricks/*` rows, not Venice additions.**

The first ~40 visible diff hunks are all `databricks/databricks-claude-*` and `databricks/databricks-gemini-*` rows being modified in two pattern classes:

1. **Floating-point representation churn:** `1.5000020000000002e-05` → `1.500002e-05`, `2.9999900000000002e-06` → `2.99999e-06`, `7.500003000000001e-05` → `7.500003e-05`, `1.5000020000000002e-05` → `1.500002e-05` (replicated across `databricks-claude-3-7-sonnet`, `databricks-claude-opus-4`, `databricks-claude-opus-4-1`, `databricks-claude-sonnet-4`, `databricks-claude-sonnet-4-1`, `databricks-claude-sonnet-4-5`, etc.). These look like a JSON serializer round-trip (`json.dump(json.load(...))` with a different float repr) rather than an intentional pricing change. The new values are mathematically equivalent (`1.500002e-05` ≡ `1.5000020000000002e-05` to within float64 precision), but the diff noise hides whether any *real* price change snuck in.

2. **Punctuation churn in `metadata.notes`:** `"Input/output cost per token is dbu cost * $0.070. Number provided for reference, ..."` → `"Input/output cost per token is dbu cost * $0.070 Number provided for reference, ..."` (period after `$0.070` deleted). Replicated across every `databricks-*` row. Looks like accidental find-and-replace damage.

3. **Capability-flag drops:** `"supports_max_reasoning_effort": true` removed from `claude-sonnet-4-20250514` (and presumably others) at `:9480` and `:9514`. This is a real semantic change — downstream consumers that branch on `model_info.supports_max_reasoning_effort` will silently flip behavior. If this is intentional (Anthropic deprecated the field?), the PR should say so; if it's accidental round-trip damage from the Venice ingest, it's a quiet regression.

**The PR title says "add Venice AI models and token prices to providers" but the actual diff modifies hundreds of unrelated rows.** A reviewer cannot extract the Venice-only intent from this diff without a side-by-side comparison or a `git log -p -- model_prices_and_context_window.json` filter. This is a **bulk-edit hygiene** failure: the PR violates the principle that pricing-data PRs should touch only the rows being added/changed.

## What needs to happen before merge

1. **Split the PR.** Move all `databricks/*` and `claude-*` row modifications to a separate "data-file regeneration" PR with explicit "this PR is no-op pricing changes from a JSON serializer round-trip" framing, and verify the `supports_max_reasoning_effort` drops are intentional (or revert them). Keep this PR focused on Venice additions only.
2. **Verify the float-repr changes are no-ops.** A small Python script (`abs(old_val - new_val) < epsilon` per row) confirming all `databricks-*` numerical changes are within float64 representation noise would convert this from "trust the serializer" to "verified equivalent."
3. **Confirm the period-deletion in metadata.notes is intentional.** If it's an editor auto-fix for "two-sentence" linting, that's fine but should be in a separate cleanup PR.
4. **Confirm the `supports_max_reasoning_effort: true` drop is intentional and tracked in a CHANGELOG entry.** If consumers branch on this flag, the silent flip is a behavior-changing release-note item that's currently invisible.

## Verdict

**needs-discussion.** The Venice AI additions themselves look fine and follow the standard model-entry shape, but the diff is contaminated with hundreds of unrelated `databricks/*` row modifications that look like JSON-serializer round-trip artifacts plus at least one capability-flag drop (`supports_max_reasoning_effort`) that's a real semantic change. The PR cannot be reviewed in this shape — split out the Venice-only changes, verify the data-regeneration arm is no-op, and confirm the capability-flag drop is intentional before re-requesting review.
