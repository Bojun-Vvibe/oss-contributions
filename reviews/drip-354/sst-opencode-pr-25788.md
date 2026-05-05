# sst/opencode PR #25788 — fix(session): distinguish malformed known-tool input from unknown tools

- Repo: `sst/opencode`
- PR: https://github.com/sst/opencode/pull/25788
- Head SHA: `39e9ca8c5a5260210729eb1b3ded723e5fb27801`
- Size: +133 / -20 across 5 files (1 src, 1 tool def, 3 tests)

## Summary

The PR factors the inline `experimental_repairToolCall` callback in
`packages/opencode/src/session/llm.ts` into an exported pure helper
`repairToolCallFailure` (llm.ts:443–483) and teaches the synthetic
`invalid` tool to surface two distinct shapes — `known_tool_invalid_input`
vs `unknown_tool` — instead of one undifferentiated "arguments are
invalid" message. `packages/opencode/src/tool/invalid.ts` (lines 4–8,
14–25) adds optional `type` and `hint` fields to the `Parameters`
schema and uses them to pick the title (`"Invalid Tool Input"` vs
`"Unknown Tool"`) and append a hint line when present.

## What I like

- Pulling the repair logic out of the inline callback makes it
  unit-testable; the three new tests in
  `test/session/llm.test.ts:33-86` cover the casing-repair branch,
  the `known_tool_invalid_input` branch, and the `unknown_tool`
  branch with concrete payload assertions (lines 158–169, 178–185).
- The schema change in `tool/invalid.ts:4-8` keeps `type` and `hint`
  *optional*, so older agent traces that only carry `{tool, error}`
  still parse — confirmed by the snapshot diff in
  `test/tool/__snapshots__/parameters.test.ts.snap` (the
  `required` array is unchanged at `["tool", ...]`).
- `metadata.tool` is now populated (`invalid.ts:19-22`), which makes
  downstream UI / telemetry able to pivot on the offending tool name
  without parsing the message body.
- The hint text in `llm.ts:466` is genuinely actionable:
  > "The tool name is valid, but the input was malformed or
  > truncated. Retry with valid input or split large operations into
  > smaller chunks."
  This nudges models toward the right recovery instead of the
  generic "arguments are invalid" they used to get.

## Nits / discussion

1. **Lower-cased name + invalid input still routes to `unknown_tool`.**
   In `llm.ts:457-463` the `known` check is
   `tools.has(original) || tools.has(lower)`. If the model sends
   `"Write"` with malformed input *and* the tool set only contains
   `"write"`, the code earlier (lines 449–456) already returns the
   case-repaired call — so this case is fine. But if `tools` contains
   only `"Write"` and the model sent `"write"` with malformed input,
   `tools.has("write")` is false, `tools.has("write")` (lower of
   "write") is also false, and you fall through to `unknown_tool`.
   That's a regression vs the old behavior, which only ever lower-
   cased one direction. Probably acceptable (real tool registries are
   lower-case) but worth a comment.

2. **`title` capitalization.** `"Invalid Tool Input"` vs the original
   `"Invalid Tool"` is a UI-facing string change. If anything in the
   web/desktop renderer pattern-matches on the old title, it'll break
   silently. A grep for `"Invalid Tool"` across `packages/desktop`
   and `packages/web` would confirm.

3. **Type narrowing.** `repairToolCallFailure` is generic over
   `T extends { toolName: string; input?: string }` but the branches
   that mutate `input` cast back with `as T` (lines 451, 469, 477).
   If a future caller passes a `T` whose `input` is required and
   non-string-typed, the cast hides it. Minor.

4. **No test for the lowercase + malformed-input edge case.** The
   three new tests cover the happy paths cleanly, but a fourth test
   `toolName: "Write"` + malformed input + `toolNames: ["write"]`
   would lock in the precedence between casing-repair and
   invalid-input classification.

## Verdict

**merge-after-nits** — drop a one-liner clarifying the
casing-vs-classification precedence (or add the fourth test), and
sanity-check the title-string change doesn't break a desktop
renderer pattern. Logic is clean and the test coverage on the new
helper is real.
