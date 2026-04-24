# PR-26393 — feat: bedrock guardrail input roles filter

[BerriAI/litellm#26393](https://github.com/BerriAI/litellm/pull/26393)

## Context

Adds `experimental_guardrail_input_roles: Optional[List[str]]` to
`BaseLitellmParams`, plumbs it through `guardrail_initializers.
initialize_bedrock` into `BedrockGuardrail.__init__`, and uses it
inside `convert_to_bedrock_format` to filter the messages forwarded
for `INPUT` validation. Use case from the linked issue: operators
want to validate only `user` turns (not the system prompt or
assistant turns) against AWS Bedrock guardrails — system prompts
contain operator-authored instructions that almost always trip a
guardrail like "no instructions to ignore prior content".

Two carve-outs:
1. The filter only applies when `source == "INPUT"`. `OUTPUT`
   validation is untouched (test
   `test_convert_to_bedrock_format_input_roles_does_not_affect_
   output` asserts this).
2. If filtering empties the message list, `convert_to_bedrock_format`
   returns an empty `BedrockRequest` early, and
   `make_bedrock_api_request` short-circuits before
   `_prepare_request` (asserted by
   `test_make_bedrock_api_request_skips_http_call_when_all_messages_
   filtered`).

## Strengths

- **Naming.** `experimental_` prefix is appropriate — operators
  reading the YAML config will know not to pin production behavior
  on this. The field's docstring also explicitly says "currently
  wired for Bedrock guardrails; silently ignored by other
  providers", which is the right honesty for a partial impl.
- **Empty-content guard is correct.** AWS Bedrock Guardrails will
  400 on a request with `content: []`. The new `if not
  bedrock_request_data.get("content"): return` short-circuit in
  `make_bedrock_api_request` saves a billable Bedrock call and
  prevents a confusing 400 from surfacing as a guardrail
  failure to the end user.
- **Test coverage is thorough for the size of the change.** 5 new
  tests: filter happy-path, no-filter passthrough, OUTPUT not
  affected, empty-after-filter returns empty `BedrockRequest`,
  and the integration-shaped test that asserts `_prepare_request`
  is not called.
- **The unrelated test fix in `test_guardrail_endpoints.py`**
  (`mock_convert.return_value = {"source": "INPUT", "content":
  [{"text": {"text": "test"}}]}` instead of empty `content: []`)
  is needed precisely because of the new short-circuit guard —
  the previous mock was implicitly relying on `make_bedrock_api_
  request` proceeding past empty content. Good that this slipped
  out as a test fix in the same PR rather than leaving a flake.

## Concerns / risks

- **Silent-skip behavior on empty-after-filter is a policy
  decision, not a bug fix.** If an operator configures
  `experimental_guardrail_input_roles=["user"]` and a request
  arrives with only system messages (e.g. a tool-only turn with no
  fresh user input), the guardrail is effectively *bypassed* —
  not flagged, not logged, just skipped. For some threat models
  this is the right default; for others (e.g. "any request to the
  proxy must hit the guardrail") it's a bypass primitive. At
  minimum, log at INFO when this short-circuit fires so operators
  can audit it.
- **Role match is exact-string, not normalized.** `m.get("role")
  in self.experimental_guardrail_input_roles` is case-sensitive
  and doesn't handle aliases (`"developer"` vs `"system"`,
  `"tool"` vs `"function"`). A configuration of `["User"]` will
  silently match nothing because OpenAI's wire format lowercases
  the role. Normalize both sides to lowercase before comparison,
  and document supported role names in the field's `description`.
- **No validation that listed roles are recognized.** If an
  operator typos `"usr"` instead of `"user"`, every request gets
  empty-after-filter and the guardrail is silently bypassed
  forever. Either fail loud at config-load time (cross-reference
  against a known-roles set) or warn-and-fall-through at request
  time.
- **Filter runs *only* for Bedrock.** The field lives on
  `BaseLitellmParams`, suggesting it could later be honored by
  other guardrail providers, but the docstring already confesses
  it's silently ignored elsewhere. Operators reading the config
  may assume "this filters all guardrails" and be surprised when
  a Presidio/PII guardrail still sees their system prompt. The
  `description` is honest but a config-load warning when the field
  is set on a non-Bedrock guardrail would prevent the surprise.
- **`messages = [m for m in messages if m.get("role") in ...]`
  rebinds the parameter.** Harmless here but means downstream
  code in the same function block that expects the original
  message list (if any is added later) gets the filtered view.
  Use a local name (`filtered = [...]`) to make the boundary
  explicit.
- **No streaming-path test.** This PR only covers the non-streaming
  `make_bedrock_api_request`. Bedrock guardrails also run in the
  streaming sse path (`make_bedrock_streaming_api_request`, judging
  by sibling PR #26388 in the same window). Worth a follow-up
  asserting the role filter applies there too, or a comment that
  it deliberately doesn't.

## Verdict

**Approve with caveats.** Useful operator knob with sensible
plumbing and good test coverage. Before merge: lowercase-normalize
role comparison, log at INFO when an empty-after-filter short-
circuit bypasses the guardrail (this is a security-adjacent
silent default), and either validate roles at config-load or warn
on unknown roles. Streaming-path coverage can ship as a follow-up.
