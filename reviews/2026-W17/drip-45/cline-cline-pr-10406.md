# cline/cline #10406 — docs: Add FuturMix AI Gateway setup guide

- **PR:** https://github.com/cline/cline/pull/10406
- **Head SHA:** `55fad25f97a9242beafe244baa720ade260e8053`
- **Files changed:** 2 — `docs/docs.json` (+1) and new `docs/provider-config/futurmix.mdx` (+44)

## Summary

Adds a new docs page describing how to point cline at FuturMix (a third-party OpenAI-compatible aggregator). Registers the new page in `docs/docs.json`. No code changes — pure documentation, configures cline as an OpenAI-compatible client against `https://futurmix.ai/v1`.

## Line-level call-outs

- `docs/docs.json:169` — the new entry `provider-config/futurmix` is alphabetically inserted between `fireworks` and `gcp-vertex-ai`. Correct ordering.
- `docs/provider-config/futurmix.mdx:1-4` — frontmatter looks fine; description mentions "22+ AI models" and "99.99% SLA" — both vendor marketing claims. cline's docs already host similar pages for other gateways (OpenRouter, etc.) so this isn't out of pattern, but it would be good practice to either (a) drop the SLA/numbers (which date instantly), or (b) link to a vendor source page that establishes them. As written, the docs commit becomes stale the day FuturMix changes their SLA tier.
- `:18-21` — the model list ("Claude 3.7 Sonnet", "Claude 3.5 Sonnet", "GPT-4o", "o1", "Gemini 2.0 Flash", "DeepSeek-V3", "Llama 3.3 70B", "Llama 3.1 405B") freezes a snapshot of model availability that will rot fast. Recommend trimming to "leading models from Anthropic, OpenAI, Google, DeepSeek, Meta, etc." and linking the live model list — which the doc already does on line 23. The enumerated list adds maintenance burden with little reader value.
- `:34` — the example model ID `claude-3-7-sonnet-20250219` should be sanity-checked — Anthropic's date-stamped IDs change, and a wrong ID copied from docs into a config file produces a confusing 404. Consider showing only the family name (`claude-3-7-sonnet`) and noting "see vendor for current dated IDs", or pointing readers at FuturMix's `/models` endpoint.
- `:39-44` — "Tips and Notes" includes `Cost Effective: Typically 20-30% cheaper than comparable gateway services like OpenRouter` — this is a comparative competitive claim about a third party (OpenRouter). Even if true, embedding it in cline's own docs creates an editorial-neutrality problem and could be challenged by the named competitor. Strongly recommend dropping the specific percentage and the OpenRouter comparison; "competitive pricing — see vendor pricing page" is plenty.
- General: no screenshots, no `.env` example. Other provider pages in cline often include a screenshot of the settings UI; not blocking, but worth matching the existing doc style.
- General: this is a single contributor adding their own service. The PR needs maintainer judgement on the broader question of "should every aggregator get a dedicated docs page or should they share one OpenAI-compatible meta-page?" — that's a policy call, not a code review item.

## Verdict

**request-changes**

## Rationale

Mechanically the PR is fine — the JSON entry is in the right place and the MDX renders. But the page contains two patterns that are reasonable to push back on before merge: (1) a marketing-style competitor comparison ("20-30% cheaper than OpenRouter") which has no place in a neutral docs site and could draw legitimate complaints, and (2) date-stamped model IDs and feature lists that will rot within a quarter. Trim the comparison claim, drop the enumerated model list in favour of the live link the page already provides, and this is merge-as-is. The underlying "support FuturMix" intent is uncontroversial and matches existing precedent for other gateways.
