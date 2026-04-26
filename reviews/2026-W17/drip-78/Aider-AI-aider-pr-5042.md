# Aider-AI/aider PR #5042 — docs: add QuickSilver Pro as an LLM provider option

- **PR:** https://github.com/Aider-AI/aider/pull/5042
- **Head SHA:** `2b0705273b10a78dcb24b90e396972b1605cf027`
- **Files:** 1 (+37 / -0)
- **Verdict:** `merge-after-nits`

## What it does

Adds `aider/website/docs/llms/quicksilverpro.md`, a 37-line provider docs
page mirroring the structure of the existing `openrouter.md` / `groq.md` /
`deepseek.md` pages. QuickSilver Pro is presented as an OpenAI-compatible
endpoint exposing DeepSeek V3, DeepSeek R1, and Qwen3.5-35B-A3B. The page
shows env-var setup (`OPENAI_API_BASE` + `OPENAI_API_KEY`) for both
Mac/Linux and Windows, then three example `aider --model openai/<id>`
invocations.

## Specific reads

- `quicksilverpro.md:1-4` — Jekyll front-matter `parent: "Connecting to LLMs"` + `nav_order: 450`. Slot 450 sits between existing providers; should be sanity-checked against the current sidebar order to avoid an out-of-sequence entry.
- `quicksilverpro.md:9-10` — claims DeepSeek V3, DeepSeek R1, Qwen3.5-35B-A3B. Note: "Qwen3.5-35B-A3B" is an unusual identifier (Qwen 3.5 series has not announced a 35B-A3B MoE variant publicly as of writing). Worth verifying with the QuickSilver Pro provider that this model name is real and stable, or trim to a generic "Qwen3 family" reference.
- `quicksilverpro.md:14` — `{% include install.md %}` reuses the canonical install block. Good — no copy-paste drift.
- `quicksilverpro.md:21-30` — the env-var pattern (`OPENAI_API_BASE` + `OPENAI_API_KEY`) is correct for OpenAI-compatible providers and matches the pattern used in sibling pages.
- `quicksilverpro.md:33-37` — three example invocations (`openai/deepseek-v3`, `openai/deepseek-r1`, `openai/qwen3.5-35b`). The `openai/<non-openai-model>` prefix is the standard OpenAI-compatible escape hatch in aider. Note: token counting and context-window detection for non-OpenAI models routed via the `openai/` prefix may fall back to GPT-4 defaults — sibling pages typically include a one-sentence caveat about this.
- The page is purely additive — no other docs are edited, no sidebar JSON updated. If aider's sidebar is auto-generated from `nav_order`, slot 450 should land it correctly; if the sidebar is hand-curated, an additional file edit is required (skim `_data/sidebars/` or similar).

## Nits before merge

1. **Verify `Qwen3.5-35B-A3B` is real** — that specific model name doesn't match any publicly-announced Qwen release (Qwen 3.5 has not been confirmed; the closest real model is `Qwen3-30B-A3B` or `Qwen3-235B-A22B`). If the provider markets it under a different real name, fix the model ID. If the contributor is confusing model versions, get a screenshot of the provider's model catalog before merge.
2. **Add the token-count/context-window caveat** consistent with `openrouter.md` style: one sentence warning that the `openai/<id>` prefix may give imprecise token counts for non-OpenAI models.
3. **Sidebar/nav verification**: confirm `nav_order: 450` lands the page in the intended position (not buried after a "footer" group). Quick `grep -r "nav_order" aider/website/docs/llms/` to compare against neighbors.
4. **Optionally trim the marketing tone**: the existing provider pages (groq, deepseek) tend to be neutral and short. The submitted page is already in that style — good. No action needed if it stays terse.
5. **Add a one-line note** about whether the provider supports streaming + tool-calling (aider's autonomous coder loop relies on both). If either is missing, document the limitation up front; users hit it 30 seconds in otherwise.
6. **No `_redirects` / no link from the provider index page**: confirm the LLMs index (`aider/website/docs/llms.md` or similar) auto-includes children, otherwise this page won't be discoverable from the sidebar root.

Docs-only, low-risk, neutral tone. Merge after the model-name sanity-check.
