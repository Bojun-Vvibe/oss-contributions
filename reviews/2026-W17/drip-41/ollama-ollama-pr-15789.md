# ollama/ollama PR #15789 — openai: map responses reasoning effort to think

- **URL:** https://github.com/ollama/ollama/pull/15789
- **Head SHA:** `2c3f76d00b43b9dd5c7d15081bd36f1f4d3b8a92`
- **Files touched:** 2 (`openai/responses.go`, `openai/responses_test.go`)
- **Verdict:** `merge-as-is`

## Summary

Maps OpenAI Responses API `reasoning.effort` values onto Ollama's internal
`api.ChatRequest.Think`. Adds explicit handling for `none` (think
disabled), `low|medium|high|max` (preserved verbatim so renderers that
care can branch on it), and a typed validation error for anything else.

## Specific references

- `openai/responses.go:528-537` — switch on `effort` builds
  `*api.ThinkValue`. `none` becomes `Value: false`; named tiers pass
  through as strings. Empty string short-circuits to `nil` (no override),
  preserving prior behaviour for clients that don't set the field.
- `openai/responses.go:567` — `Think: think` plumbed onto the returned
  `ChatRequest`. Clean addition, no other request fields touched.
- `openai/responses_test.go:+80` — table covers `unset`, `low`, `medium`,
  `high`, `max`, `none`, and an invalid value asserting the error
  message. Test parity with the production switch is exactly what you
  want for value-mapping code.

## Reasoning

Tight, surgical change. The `max` tier is preserved as a string rather
than collapsed into `high`, which matches the PR description's intent
("renderers that need it"). Error message lists all valid values, which
is the right UX for an API surface error.

No nits. Merge.
