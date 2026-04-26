---
pr: 5070
repo: Aider-AI/aider
sha: 3028be8b7a0c7bc5970609bb51593f4bd70845b6
verdict: merge-after-nits
date: 2026-04-26
---

# Aider-AI/aider #5070 — docs: add FuturMix LLM provider documentation

- **URL**: https://github.com/Aider-AI/aider/pull/5070
- **Author**: Gzhao-redpoint (Gerry Zhao)
- **Head SHA**: 3028be8b7a0c7bc5970609bb51593f4bd70845b6
- **Size**: +77/-0, 1 file (`aider/website/docs/llms/futurmix.md`)

## Scope

New LLM provider docs page for FuturMix, an OpenAI-compatible AI gateway. Pure documentation — no code, no config, no router changes. The page registers under the `Connecting to LLMs` parent in Jekyll/just-the-docs front matter (`nav_order: 500`) and explains the 3-step setup: install aider, set `OPENAI_API_BASE` + `OPENAI_API_KEY`, run `aider --model openai/<model>`.

## Specific findings

- **The setup is just OpenAI-compatible base-URL routing.** Lines 18-26 of the new file set `OPENAI_API_BASE=https://futurmix.ai/v1` and `OPENAI_API_KEY=<key>` and that's it. This is the same pattern aider already documents for OpenRouter, Together, DeepSeek, Groq, and a half-dozen other gateways under `aider/website/docs/llms/`. So adding another gateway page is low-friction; the precedent is clear.

- **Model list at lines 39-58 will go stale fast.** The page hard-codes `claude-opus-4-20250514`, `claude-sonnet-4-20250514`, `gpt-4-turbo`, `gpt-4o`, `o1`, `gemini-2.0-flash-exp`, `gemini-1.5-pro`, etc. By the time this lands, several of these are already deprecated upstream (e.g. `claude-3-5-haiku-20241022` is past its sunset window in 2026; `o1` has been superseded by `o3`/`o4` family). **Nit:** trim this section to a single paragraph saying "see FuturMix's docs for the live model list," or at minimum remove the dated date-stamped model IDs and keep only family names. The stale-list pattern is exactly why the OpenRouter page in this same directory tree avoids enumerating models.

- **`--model openai/claude-sonnet-4-20250514` example at line 32 is misleading.** Routing a Claude model through `openai/`-prefixed namespace works because aider maps `openai/<id>` to "use OpenAI-style HTTP transport with this model name" — but the prefix doesn't tell aider that the model is actually Claude under the hood, which means token-counting, context-window estimation, and streaming-format hints all fall back to OpenAI defaults. For most use cases that's fine, but if a user ends up debugging a context-overflow miscount they'll be confused why aider thinks a 200K-context Claude model only has 8K. **Nit:** add a sentence noting this caveat or referencing the existing aider docs on `--map-tokens` / model-info overrides.

- **No claims about caching, no claims about tool-calling support.** The page sticks to base-URL routing, which is honest — gateways often differ from upstream providers on prompt-cache headers, parallel tool-calls, structured-output mode, etc. Better to undersell than to promise behavior that varies per upstream model.

- **Marketing flavor in the "Benefits" block (lines 70-76)** — "99.99% SLA," "competitive pricing," "drop-in replacement." A maintainer may want to trim this; aider's other gateway pages (e.g. `openrouter.md`, `groq.md`) tend to be neutral and instruction-focused rather than vendor-promotional. **Nit:** drop the bullet list or move it to a one-line summary at the top. Keeping it isn't a blocker — every gateway page in this directory has *some* marketing flavor — but the four-bullet format is more sales-deck than docs.

- **`nav_order: 500`** — large enough number that the page sorts toward the bottom of the parent's TOC. Consistent with how new providers get slotted in over time; not a conflict with any other page's `nav_order`.

- **Author looks first-time-ish.** `Gzhao-redpoint` doesn't appear in the recent commit log. The page follows the existing convention closely (front matter, install include, env vars, `aider --model` examples, references), so this is a low-friction onboard. The likely-affiliation ("redpoint" suffix) suggests a vendor employee submitting their own product's docs — that's normal and accepted in aider's gateway ecosystem, but maintainer should confirm the upstream service actually works as described before landing (try `curl https://futurmix.ai/v1/models` with a real key to verify it's a real, reachable OpenAI-compatible endpoint).

## Risk

Negligible — pure docs. Worst-case is a stale or vendor-overstated page that future contributors clean up. No code paths affected.

## Verdict

**merge-after-nits** — three small ones:
1. Trim the hard-coded model list (lines 39-58) to family names + "see vendor docs for live list."
2. Add a one-sentence caveat about `openai/<non-openai-model>` token-count/context behavior, since the example uses Claude through this prefix.
3. Optional: trim the "Benefits" block (lines 70-76) to match the neutral tone of `openrouter.md` / `groq.md`.

If the maintainer is willing to land it as-is and clean up the model list in a follow-up, that's also reasonable — vendor docs pages are routinely landed and trimmed later.

## What I learned

Hard-coded model lists in docs are the single most common source of long-term staleness in LLM-tooling repos. The robust shape is "redirect to vendor's `/v1/models` endpoint or vendor docs page" — every project's gateway docs that enumerate models gets a model-deprecation churn PR every 3-4 months. Worth treating "no enumerated model list" as a soft style rule for gateway docs.
