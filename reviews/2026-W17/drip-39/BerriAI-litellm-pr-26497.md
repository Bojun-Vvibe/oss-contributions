# BerriAI/litellm #26497 — fix(chatgpt): preserve text and parallel_tool_calls in responses

- **Repo**: BerriAI/litellm
- **PR**: [#26497](https://github.com/BerriAI/litellm/pull/26497)
- **Head SHA**: `c5040a84707515f4aab6346eab19e9c70abb2007`
- **Author**: HeMuling
- **State**: OPEN (+56 / -0)
- **Verdict**: `merge-as-is`

## Context

The `chatgpt/...` provider's Responses-API transform was
filtering its allow-list of forwarded params and silently
dropping `text` (structured-output schemas) and
`parallel_tool_calls`. End users setting `response_format` /
`text.format.json_schema` against `chatgpt/gpt-5.2-codex` got
freeform text back instead of the requested schema. Fix is two
allow-list entries plus tests.

## Design

`litellm/llms/chatgpt/responses/transformation.py:96-105` adds
`"parallel_tool_calls"` and `"text"` to the supported-params set
inside `transform_responses_api_request`. That's the entire
behavioural change — eight characters of new content per line,
two lines.

Tests at
`tests/test_litellm/llms/chatgpt/responses/test_chatgpt_responses_transformation.py`:

- New `test_chatgpt_preserves_text_param` (lines 109-145) is a
  three-case parametrize over the realistic `text.format`
  shapes: `json_schema` (with `strict: True`), `json_object`,
  and bare `text`. Asserts the param is forwarded verbatim and
  that the existing streaming/reasoning-include defaults aren't
  disturbed.
- `test_chatgpt_preserves_parallel_tool_calls` (lines 147-159)
  is the boolean-false case to prove the value isn't being
  truthy-coerced and dropped.
- The existing `test_chatgpt_drops_unsupported_responses_params`
  (lines 184-208) is extended with both new params in the
  "preserved" assertions, so future regressions will fail two
  tests instead of one.

## Risks

- **Param ordering** in the set literal at lines 96-105 is the
  only nit: `parallel_tool_calls` lands between `include` and
  `tools`, while `text` lands between `reasoning` and
  `previous_response_id`. The set is unordered at runtime so
  this is purely cosmetic, but a strictly-alphabetical or
  group-by-purpose ordering would make future additions less
  prone to merge conflicts.
- **No schema validation of the `text` param.** Whatever the
  caller passes goes downstream verbatim. That's consistent with
  how the other forwarded params work (`tools`, `reasoning`),
  so it's the right default — but worth noting that a malformed
  `text.format.json_schema` will surface as an OpenAI 400
  rather than a litellm-side error. Acceptable trade.
- **No CHANGELOG / release-note line** in the diff. For a
  user-visible structured-output fix, a note in the next release
  is worth adding before merge — it'll save future debugging
  time when someone bisects "when did `chatgpt/gpt-5.2-codex`
  start respecting `response_format`?".

## Suggestions

- Add a one-liner to the next release notes referencing this PR.
- (Optional) sort the supported-params set alphabetically so
  future allow-list additions don't drift into random spots.

## What I learned

Allow-list-based param forwarding is brittle in exactly this
way: every new OpenAI Responses API field needs a corresponding
allow-list edit, and the failure mode (silent drop, no error,
just bad output) is the worst possible kind. A complementary
approach — allow-list of *known-bad* params with a passthrough
default — would surface unknown fields to the upstream API
where validation lives. That's a bigger refactor than this PR
should attempt; this fix is correct as-is.
