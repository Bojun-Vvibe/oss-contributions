# BerriAI/litellm#26508 — Litellm oss staging 04 25 2026
**SHA**: `334aedf2` · **Author**: krrish-berri-2 · **Size**: +21/-6 across 4 files

## What
Three independent fixes bundled as a staging cherry-pick: (1) drop the
unused `MAX_SIZE_PER_ITEM_IN_MEMORY_CACHE_IN_KB` constant from
`litellm/constants.py`; (2) make the Cloudflare chat transformer accept
either `result.response` or `result.response_text` (newer Cloudflare
models like Nemotron emit `response_text`); (3) add Z.AI (Zhipu AI) as
a first-class entry in the dashboard provider dropdown with logo
mapping and a `zai/glm-4.5` placeholder.

## Specific cite
- `litellm/llms/cloudflare/chat/transformation.py:150-152` is the
  load-bearing change:
  `result.get("response") if result.get("response") is not None else result.get("response_text", "")`.
  Note the deliberate `is not None` rather than truthiness — this
  preserves empty-string responses from the older API rather than
  falling through to `response_text`. Correct, but non-obvious; worth
  an inline comment.
- `ui/litellm-dashboard/src/components/provider_info_helpers.tsx:106,214,371`
  add the `ZAI` enum member, `provider_map` entry, and placeholder
  branch. The display name `"Z.AI (Zhipu AI)"` matches what the
  backend already returns from `/public/providers` (per the regression
  test comment referencing issue #25482).
- `provider_info_helpers.test.tsx:71-78` is a real regression test —
  asserts `getProviderLogoAndName("zai").displayName === Providers.ZAI`,
  which would have failed before the mapping was added.
- The constant deletion (`constants.py:412-414`) is fine in isolation
  but should be grep-confirmed unused across the repo and any plugin
  surface; if external code imports it the PR breaks them silently.

## Verdict: merge-after-nits
The three changes are independently sound, but bundling unrelated
fixes in a single staging PR makes bisecting future regressions
harder. Two nits: (1) add an inline comment at the `is not None`
guard in `transformation.py` explaining why empty-string is
preserved, and (2) confirm via grep that
`MAX_SIZE_PER_ITEM_IN_MEMORY_CACHE_IN_KB` has no external importers
before deletion.
