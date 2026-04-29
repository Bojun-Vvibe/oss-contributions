# sst/opencode#24919 — `feat(provider): convert reasoning_effort to thinking_config for Gemini`

- PR: https://github.com/sst/opencode/pull/24919
- Head SHA: `fe407e418337543e49cd546280dfa585ab6828bd`
- Author: darthbeeius

## Summary of change

Extends the `apiMode === "chat"` body-rewriter introduced in #24914 with a
third pass: when the outbound body carries `reasoning_effort` and
`body.model` contains the substring `"gemini"`, swap it for a
`thinking_config: { thinking_level: <level> }` shape. Lives in the
`fetchFn` interceptor in `packages/opencode/src/provider/provider.ts`
around lines 1534–1574 (touch is the new branch added inside the
`apiMode === "chat"` guard, alongside the `max_tokens →
max_completion_tokens` rename and `$ref` resolution).

## Specific observations

- `packages/opencode/src/provider/provider.ts:1551-1559` — the level map
  is `{ none: "THINKING_LEVEL_UNSPECIFIED", low: "LOW", medium: "LOW",
  high: "HIGH" }`. Folding `medium → LOW` is a real semantic loss: a
  user who explicitly picked `reasoning_effort=medium` in their config
  silently gets `LOW` thinking on Gemini, with no log line. There are
  also Gemini-side levels not represented (`MEDIUM`, `DYNAMIC`,
  `UNLIMITED`) — at minimum `medium → "MEDIUM"` should be tried, and a
  fallthrough `?? "LOW"` is the wrong default to silently use for an
  unknown effort string.
- Same lines: detection is `body.model.includes("gemini")`. This is a
  loose substring that will fire on `"gemini-1.5-flash"`,
  `"my-gemini-tuned"`, and `"openai/o1-non-gemini-bench"` alike — and
  also misses any provider that rebrands the model id (e.g. routed via a
  gateway under a non-`gemini` slug). A model-family resolver (or at
  least a `/^gemini[-/]/` anchored regex on the path tail) would be
  closer to intent.
- The rewrite happens unconditionally when both conditions hit, even if
  the user explicitly set `body.thinking_config` already (e.g. via a
  custom `options` blob). The translation should be a no-op when
  `thinking_config` is already present.
- `THINKING_LEVEL_UNSPECIFIED` for `none` is technically the protobuf
  default — sending it explicitly may or may not be equivalent to
  omitting the field, depending on which Gemini surface is on the other
  end of the gateway. Worth double-checking against the actual proxy
  vendor behaviour rather than treating "unspecified" as
  "no thinking".
- No new test coverage added in this PR for the Gemini-specific branch
  — the three tests appended in the test file (lines 2715–2844 of
  `packages/opencode/test/provider/provider.test.ts`) cover the
  `apiMode` plumbing from #24914 but not the `reasoning_effort →
  thinking_config` translation. A unit test that constructs a body with
  `model: "gemini-2.0-flash"` and `reasoning_effort: "high"` and asserts
  the translated body would catch the `medium → LOW` issue immediately.

## Verdict

`request-changes` — the level map and the substring model match are
both bugs in the steady state, and the rewrite needs a "skip if
`thinking_config` already set" idempotency guard before it lands.
