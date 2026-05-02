# BerriAI/litellm #27042 — docs: clarify ollama_chat usage to avoid empty responses

- URL: https://github.com/BerriAI/litellm/pull/27042
- Head SHA: `65c538078627482e83198028fa8a7cb151edecfe`
- Author: @douminik
- Stats: +4 / -0 across 1 file

## Summary

Adds a 3-line clarifying note to the Ollama provider docs warning that
`ollama/...` may return empty `{}` payloads in chat-style flows (TUI /
interactive agents) and recommending `ollama_chat/...` for stable
behavior.

## Specific feedback

- `docs/my-website/docs/providers/ollama.md:15-18` — note is inserted
  inside the existing `:::tip` (the line above already says "We recommend
  using ollama_chat for better responses"). Placement is correct; the
  new note expands on the *why* (TUI / interactive flows specifically)
  rather than restating the recommendation.
- Wording is clear: "may lead to unexpected behavior such as empty
  responses (`{}`) or missing outputs" is concrete enough that someone
  hitting the bug can grep for it.
- No code touched, no schema change, no test risk.
- Nit: a one-line example showing `ollama_chat/llama3` vs `ollama/llama3`
  in a code fence would make the contrast scannable, but that's a
  preference, not a blocker.

## Verdict

`merge-as-is` — pure docs, addresses a real and frequently-hit pitfall.
