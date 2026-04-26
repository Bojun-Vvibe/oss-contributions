# BerriAI/litellm #26530 — chore(staging): roll oss_staging_04_25_2026 into internal staging (output_config fix + 4 upstream sync fixes)

- **Author:** mateo-berri
- **Head SHA:** `0644a1b02b43444d7e2567f965e63637557692fc`
- **Base:** internal staging (per author's PR title)
- **Size:** +1003 / -93 (17 files)
- **URL:** https://github.com/BerriAI/litellm/pull/26530

## Summary

A staging-roll-up PR that consolidates one headline fix (PR #26439:
`output_config` passthrough on Vertex Anthropic + the
`/v1/messages` → `/chat/completions` adapter) plus four already-merged
upstream sync fixes (#26506, #26401, #26419, #26385) into the internal
staging branch. The headline fix has real design choices worth
reviewing; the four sync-pulls are mechanical forwards of PRs that
already passed their own review.

## Specific findings

**Headline fix — Vertex Anthropic `output_config` passthrough:**

- `litellm/llms/vertex_ai/vertex_ai_partner_models/anthropic/output_params_utils.py:1-50`
  (new file). The author extracted `sanitize_vertex_anthropic_output_params`
  into a leaf module to break a CodeQL-flagged cyclic import through
  `..transformation`. The helper documents its behavior precisely:
  unsupported-only `output_config` is removed entirely, mixed
  configs are filtered, supported configs pass through, and non-dict
  values are dropped to avoid malformed payloads. Defining
  `VERTEX_UNSUPPORTED_OUTPUT_CONFIG_KEYS = frozenset({"effort"})` with
  the comment "add new entries here as Vertex parity drifts" is the
  right shape for this kind of provider-quirk allowlist.

- `litellm/llms/anthropic/experimental_pass_through/adapters/handler.py:31-37`
  introduces `ANTHROPIC_ONLY_REQUEST_KEYS = frozenset({"output_config"})`
  with a maintainer-facing comment explaining the consequence of
  forgetting to extend it ("400 Extra inputs are not permitted on
  non-Anthropic backends"). Good failure-mode documentation.

- `handler.py:215-218` — `extra_kwargs or {}` → `extra_kwargs if
  extra_kwargs is not None else {}` is a real semantic fix: callers who
  pass an explicit empty dict were previously getting a None-coerced
  fresh dict that broke identity-based tests of the fallback inference
  path. The follow-up comment at line 256–257 ("NOTE: extra_kwargs was
  already coerced from None to {} at the top of this method") is
  correct given the order of operations in `_prepare_completion_kwargs`.

- `handler.py:241-254` — the comment block on `excluded_keys` is
  unusually thorough and explains *both* the non-Anthropic 400 failure
  *and* the Anthropic-target silent-conflict failure ("would see the
  translated key `response_format` AND a duplicate, conflicting
  `output_config`"). This is the kind of comment that prevents the same
  bug from being reintroduced. Worth keeping.

**Greptile P1 worth confirming before merge into main:**

The PR body acknowledges that `test_vertex_ai_claude_sonnet_4_5_structured_output_fix`
flips its assertion from "stripped" to "forwarded" based on a single
data point (issue #18625, originally negative). The author's own heads-up
note: "Recommend confirming against a live Vertex Claude endpoint
before this merges further." That request is correct and should not be
waived. If Vertex still 400s on `output_format` + `tools`, the
sanitizer's `VERTEX_UNSUPPORTED_OUTPUT_CONFIG_KEYS` set must be
extended (and the docstring on `sanitize_vertex_anthropic_output_params`
already implies it would be — but the implementation currently only
filters `output_config`, not `output_format`).

**Sync-pulled fixes (riding along):**

- `litellm/integrations/arize/_utils.py` — `_set_usage_outputs` now
  handles raw OpenAI Pydantic `CompletionUsage`; this is the right
  defensive shape change since the previous dict-only assumption was
  silently dropping observability fields.
- `litellm/proxy/proxy_server.py` — `LITELLM_LOG=INFO` now correctly
  propagates to `verbose_logger`. Previous behavior was a no-op above
  WARNING, which is a legit ops paper cut.
- `litellm/constants.py:85` — `MAX_SIZE_PER_ITEM_IN_MEMORY_CACHE_IN_KB`
  duplicate-definition consolidation. Greptile P0 is a false alarm as
  the author notes; the surviving 1024 default is the correct one and
  both downstream importers resolve cleanly.
- `ui/litellm-dashboard/src/components/provider_info_helpers.tsx` —
  `zai` (Z.AI / Zhipu AI) now appears in the Add-Model dropdown; this
  is a re-fix of #25482 that regressed.

## Verdict

`needs-discussion`

## Reasoning

The headline fix is well-structured and the comments are unusually
self-aware about failure modes. **But** this PR bundles a real
behavioral change (Vertex Anthropic `output_format` passthrough)
whose key test assertion was flipped on the strength of a single
negative data point — and the author's own PR body explicitly flags
the live-endpoint validation as not done. That open verification gate,
combined with this being a 5-changes-in-one staging roll-up, makes
"merge as-is" inappropriate. The right path: confirm Vertex actually
accepts `output_format` + `tools` (or extend the sanitizer to strip
it), then merge. If Vertex rejects, the sanitizer module is already
factored to make the one-line allowlist extension trivial.

The four sync-pulled fixes are individually fine and could be split
into a separate "sync-only" PR if blocking on the Vertex confirmation
turns out to be slow.
