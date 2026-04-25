# OpenHands/OpenHands #14086 — fix: don't persist Gemini model-specific endpoint as base_url

- **PR:** https://github.com/OpenHands/OpenHands/pull/14086
- **Head SHA:** `d0c1858a07b80377e0dd5e5da2790c14bcc1cfad`
- **Files changed:** 2 — `openhands/utils/llm.py` (+7/−0), `tests/unit/utils/test_llm_utils.py` (+12/−5)

## Summary

Fixes #14028. `litellm.get_api_base()` returns inconsistent shapes across providers: for OpenAI/Anthropic/Mistral it returns a reusable base URL (`https://api.openai.com`); for `gemini/<model>` it returns the **fully-resolved request URL** (`https://generativelanguage.googleapis.com/v1beta/models/gemini-pro:generateContent`). OpenHands then persists that fully-resolved URL as `base_url` on the LLM config, and the next request appends the model path a second time, hitting `.../models/gemini-pro:generateContent/models/gemini-pro:generateContent` and 404-ing forever. Fix: detect the `:generateContent` / `:streamGenerateContent` suffix and return `None` instead, letting litellm fall back to its own default base.

## Line-level call-outs

- `openhands/utils/llm.py:185-191` — `if api_base.endswith((':generateContent', ':streamGenerateContent')): return None`. **Correct minimal fix**, but it's tightly coupled to the current Gemini URL convention. Two failure modes worth pinning down:
  1. **Future Gemini suffix drift.** If Google ever ships a new action verb (e.g. `:embedContent`, `:countTokens`, `:predict`) the suffix-detection misses it and the double-append bug returns. The robust check would be "any URL whose path segment after `/v1beta/models/` is non-empty" — i.e. detect the *shape* (model-resolved path) rather than the *verb*. Even simpler: detect `/models/` in the path and treat any URL containing a model segment as non-reusable.
  2. **Other providers with the same misshape.** `litellm.get_api_base()` is documented to return inconsistent things; the same pattern almost certainly affects Vertex AI (`/publishers/google/models/<model>:streamGenerateContent`), and possibly Cohere. The fix is Gemini-only by hard-coded suffix; please add at minimum a `# TODO` referencing the litellm upstream issue, and ideally extend the check to `:streamGenerateContent` + Vertex AI's path shape in the same PR.
- `openhands/utils/llm.py:186-189` — the inline comment is excellent: it cites the issue number, explains *what* litellm returns, *why* persisting it is wrong, and *what behaviour* the early return triggers. This is the right shape for a defensive guard against an upstream API quirk. Keep it.
- `tests/unit/utils/test_llm_utils.py:111-122` — old test asserted "Gemini returns a Google base URL" (i.e., it was asserting the **buggy** behaviour). New test inverts the assertion: `assert get_provider_api_base('gemini/gemini-pro') is None` and adds `'gemini/gemini-2.5-pro'`. Good. **Missing test:** no assertion that `gemini-2.5-flash` (or any future model name) also returns `None`. The bug is not model-specific — it's URL-shape-specific — so the test should mock `litellm.get_api_base` to return a known fully-resolved URL and assert the suffix check fires regardless of model name. As written, the test is a regression test for two specific model strings, not for the fix's invariant.
- The fix is a **client-side workaround for an upstream litellm quirk**. The right long-term fix is a litellm bug (their `get_api_base` should return a reusable base for Gemini, or at minimum document which providers return resolved vs. base URLs). The PR description doesn't link a corresponding litellm issue. Please file one and reference it in the comment so the workaround can be removed when litellm fixes it.

## Verdict

**merge-after-nits**

## Rationale

This is a correctly-diagnosed, narrowly-scoped fix for a high-impact bug ("every Gemini message returns 404"). The diagnosis is right, the comment is right, and the regression test exists. The two issues are forward-looking, not blocking: (1) the suffix list will silently rot the next time Google ships a new action verb — generalize the shape check; (2) Vertex AI almost certainly has the same bug and this PR doesn't address it. Either tighten the check to a shape-based detection now, or land this and immediately follow up with a Vertex test + extension. Either way merge-as-is is fine if the author confirms scope is "Gemini-only by intent, Vertex tracked separately"; otherwise hold for the broader fix.
