# BerriAI/litellm#26970 — feat: add Venice AI models and token prices to providers

- PR ref: `BerriAI/litellm#26970` (https://github.com/BerriAI/litellm/pull/26970)
- Head SHA: `3281f7235ba2ed553ddf6f05b5e2fc3998e845b6`
- Title: feat: add Venice AI models and token prices to providers
- Verdict: **request-changes**

## Review

The Venice AI catalog additions themselves look reasonable on their face — both
`model_prices_and_context_window.json` and the `_backup.json` mirror gain a coherent
block of `venice/*` entries (e.g. `venice/zai-org-glm-5-1` at ~line 315 of the diff,
`venice/venice-uncensored` ~line 388) with consistent shape: `litellm_provider:
"venice"`, `mode: "chat"`, sane token caps. That part is mergeable.

What blocks merge today is the **scope of the diff doesn't match the PR title**.
Beyond the Venice additions, the same patch silently:

1. Rewrites the schema-doc string for `max_tokens` at the top of the JSON file
   (`"LEGACY parameter..."` → `"max output tokens..."`) — this changes documented
   semantics for every consumer that reads the schema preface, and it doesn't
   belong in a "add Venice models" PR.
2. Adds `"max_tokens": 2000` to a long list of unrelated audio-transcription
   models (`azure/gpt-4o-transcribe`, `azure/gpt-4o-transcribe-diarize`,
   `gpt-4o-transcribe`, etc.) — possibly correct, but each of those is a
   reviewable change on its own.
3. Adds `"max_tokens": 8192` to `gemini-2.0-flash-001`, `gemini-2.0-flash-lite-001`,
   `gemini/gemini-2.0-flash-001` — same scope concern.
4. **Removes** `"supports_max_reasoning_effort": true` from
   `claude-opus-4-6-20251015` and `claude-opus-4-7-20260416` (line 9477 / 9512
   region of the diff). That is a behavioral capability bit, not a price/catalog
   entry, and removing it without justification in the description is the most
   concerning change in the diff — downstream callers reading that flag will
   silently change behavior.

Asking author to split into (a) Venice additions only, (b) the schema-doc
edit + audio model `max_tokens`, and (c) the Claude reasoning-effort capability
removal, with separate justification on (c). Once split, (a) is straightforward
`merge-as-is`.
