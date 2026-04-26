# Aider-AI/aider PR #5069 — docs: Add FuturMix AI Gateway to LLM providers

- **Repo:** Aider-AI/aider
- **PR:** [#5069](https://github.com/Aider-AI/aider/pull/5069)
- **Files:** 1 file, +77/−0 (new file `aider/website/docs/llms/futurmix.md`)

## Context

Adds a new docs page under "Connecting to LLMs" describing FuturMix
(an OpenAI-compatible AI gateway claiming 22+ models from
Anthropic/OpenAI/Google/DeepSeek/xAI behind one endpoint at
`https://futurmix.ai/v1`). The page follows the same template as
the existing per-provider docs (`openrouter.md`, `together.md`, etc.):
front-matter with `parent: Connecting to LLMs`, install snippet,
env-var export, sample `aider --model …` invocations, model list.

## What the diff actually does

**`aider/website/docs/llms/futurmix.md` (new, 77 lines)**:
- Front-matter at `:1-4` with `nav_order: 500` (a high number, so it
  sorts toward the end of the LLM provider list).
- Connection guidance at `:24-33`: export `OPENAI_API_BASE` and
  `OPENAI_API_KEY` (uses the OpenAI-compatible path, no new
  litellm provider prefix).
- Sample invocations at `:42-48`:
  - `aider --model openai/claude-sonnet-4-20250514`
  - `aider --model openai/gpt-4-turbo`
  - `aider --model openai/gemini-2.0-flash-exp`
- Curated model list at `:55-73` split into Anthropic / OpenAI /
  Google sections.
- Closing "Benefits" bulletlist at `:79-83` repeating the marketing
  claims (single API key, OpenAI-compatible, 99.99% SLA,
  competitive pricing, easy switching).

## Strengths

- **Follows the template**. Front-matter shape, `{% include
  install.md %}` shortcode, and Mac/Linux/Windows env-var pattern
  all match the existing per-provider pages, so the page slots in
  cleanly without theme work.
- **OpenAI-compatible path is the right approach** — no new
  litellm provider prefix is required because FuturMix exposes
  the `/v1` shape, so `OPENAI_API_BASE` redirection works out of
  the box. No code changes needed in aider itself.
- **Concrete invocations at `:42-48`** give the user a paste-and-go
  starting point rather than just abstract "set these env vars."

## Risks / nits

1. **Marketing claims unverified.** "99.99% SLA", "22+ AI models",
   "competitive pricing" all appear at `:14` and `:79-82` without
   citation. Aider's docs aren't the place to repeat a vendor's
   marketing copy verbatim — the maintainers will have to choose
   between (a) trimming to neutral facts ("OpenAI-compatible
   gateway exposing models from multiple providers") or (b)
   accepting the precedent that vendor-submitted docs PRs may
   carry promotional language. Recommend (a).
2. **`nav_order: 500` is suspicious.** A quick check of other
   provider pages will probably show `nav_order` values in the
   1-50 range. 500 will work (Jekyll just sorts numerically) but
   it suggests the author didn't read the conventions. Pick a
   number consistent with neighbors (probably alphabetical
   placement between `fireworks.md` and `gemini.md`).
3. **Model id `openai/claude-sonnet-4-20250514`** at `:42` is
   semantically odd — it's a Claude model with the `openai/`
   litellm prefix because the gateway is OpenAI-compatible. This
   *will* work but reads as a typo. Add a one-sentence callout:
   "Note: all models use the `openai/` prefix because FuturMix
   speaks the OpenAI API, regardless of the underlying provider."
4. **`gemini-2.0-flash-exp` is an experimental ID.** By the time
   this doc lands, the `-exp` suffix may have been replaced with
   a stable id. Either link to FuturMix's model list (already
   done at `:75`) and prune the inline list, or add a "as of
   date" timestamp.
5. **No verification that FuturMix actually works with aider.**
   Docs PRs for niche gateways have historically landed without
   anyone on the maintainer side actually testing the path. Ask
   the author to attach a screenshot or transcript of `aider
   --model openai/<some-model>` connecting successfully through
   FuturMix.
6. **The "Benefits" section at `:77-83`** is pure marketing and
   adds no operational value. Recommend deletion — the user
   already chose to read this page, the sales pitch is redundant.
7. **No mention of rate limits, error handling, or what happens
   when a model is unavailable on the gateway.** Existing
   provider pages (e.g., `openrouter.md`) typically include a
   troubleshooting hint. A "if you see X, try Y" line would
   match precedent.
8. **Author appears to be FuturMix-affiliated** (not verified in
   the metadata, but vendor-authored docs PRs are common). That's
   not disqualifying — most provider docs in `aider/website/docs/llms/`
   were originally vendor-submitted — but it does heighten the
   bar for trimming marketing language.
9. **No CHANGELOG / release-notes entry.** Adding a provider page
   typically gets a one-line mention in `HISTORY.md`. Worth
   confirming with maintainers whether this is required for docs
   PRs.

## Verdict

`request-changes`

The page is structurally correct (template followed, front-matter
valid, OpenAI-compatible path is the right integration approach)
but it reads as vendor-supplied marketing copy rather than
operational docs. Three concrete asks before merge:

1. Strip the "99.99% SLA / competitive pricing / Cost Effective"
   marketing language at `:14` and `:77-83` down to neutral
   factual statements.
2. Fix `nav_order: 500` to something consistent with neighboring
   provider pages.
3. Add a one-sentence note explaining why Claude/Gemini models
   use the `openai/` litellm prefix (because the gateway is
   OpenAI-compatible), so users aren't confused by
   `openai/claude-sonnet-4-20250514`.

Optional: attach proof-of-life (`aider` actually connecting through
FuturMix) and add a "tested as of YYYY-MM-DD" footer so the model
list has a known staleness window. Once those land it's a clean
`merge-as-is`.
