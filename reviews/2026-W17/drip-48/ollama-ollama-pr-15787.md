# ollama/ollama#15787 — api: accept "max" as a think value

- **URL**: https://github.com/ollama/ollama/pull/15787
- **Author**: ParthSareen
- **Head SHA**: `2389eddc504acdbc4be2e95a607eb20018991c3d`
- **Verdict**: `merge-after-nits`

## Summary

Extends the `ThinkValue` enum in `api/types.go` to accept the string
`"max"` alongside the existing `"high"`, `"medium"`, `"low"`. Touches
five files: `api/types.go` (validator + Bool() helper +
UnmarshalJSON), `api/types_test.go` (one new test case),
`cmd/cmd.go` (the `--think` CLI flag parser at line ~585),
`openai/openai.go` (the OpenAI-compat `reasoning_effort` →
`ThinkValue` mapper at line ~635), and `server/routes.go` (Generate
and Chat handlers map `"max"` → `"high"` for the harmony parser).

## Reviewable points

- `api/types.go:1099` and `:1133` both extend the
  `v == "high" || v == "medium" || v == "low"` chain to include
  `"max"`. Consistent across `IsValid()` and `Bool()`.
- The `UnmarshalJSON` error message at line ~1175 is updated to
  include `"max"` in the expected-values list — good.
- `server/routes.go` adds a `"max"` → `"high"` rewrite *only when*
  `shouldUseHarmony(m)` is true (lines ~378–384 in `GenerateHandler`,
  duplicated lines ~2329–2335 in `ChatHandler`). The duplication is
  unfortunate; a tiny helper `func clampThinkForHarmony(req *...)`
  would deduplicate and keep the two handlers in sync if the mapping
  ever needs to change again.
- The `openai/openai.go` change at line ~634 adds `"max"` to the
  allowlist for `reasoning_effort`. The new test
  `TestFromChatRequest_ReasoningEffort` covers `unset`, all four
  string values, `none`, and `extreme` (invalid). Coverage is solid.
- Semantic question worth resolving in the PR description: when the
  harmony parser is active, `"max"` is silently demoted to `"high"`.
  That means a user passing `think: "max"` to a harmony-backed model
  cannot distinguish it from `think: "high"` after the rewrite. If a
  future model surfaces a real `"max"` tier, this clamp will need to
  be revisited. Recommend a comment at the rewrite site noting that
  this is a forward-compat clamp, not a permanent identity.
- Single test case added in `types_test.go` for `"max"`. A symmetric
  case asserting `"max"` round-trips through `MarshalJSON` would be
  cheap insurance.

## Rationale

Clean enum extension with consistent updates across the public API
surface, the CLI flag, the OpenAI-compat layer, and the server
handlers. The harmony clamp is the one design decision worth
documenting inline. Tests cover the new value end-to-end through the
OpenAI-compat path. Mergeable after the small dedup + comment nits.
