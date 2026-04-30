# google-gemini/gemini-cli#26262 — fix: report AgentExecutionBlocked in non-interactive programmatic modes

- PR: https://github.com/google-gemini/gemini-cli/pull/26262
- Head SHA: `dedd4937c32e4f47f077e24f4e8582dddcf4d895`
- Author: cocosheng-g (Coco Sheng)
- Files: 11 changed, +488/−24
- Fixes #20859

## Context

`AgentExecutionBlocked` events (emitted when a hook or policy
short-circuits a tool execution) were silently swallowed in the
two non-interactive output modes (`OutputFormat.JSON` and
`OutputFormat.STREAM_JSON`). Operators using gemini-cli in CI
or scripted contexts would see the agent stop without any
machine-readable signal explaining *why* — the human-
readable interactive UI surfaces the block as a warning toast,
but the JSON/STREAM_JSON consumers got nothing.

## Design

Two-pronged fix mirroring the existing
warning/error/finished pattern:

**JSON mode** — `nonInteractiveCli.ts` collects a `warnings:
string[]` list. Pre-PR `JsonOutput` had no warnings field;
the diff at `output/types.ts:+1` adds it as optional, and
`output/json-formatter.ts:+5` collects + ANSI-strips warning
messages before serializing. Run-loop side at
`nonInteractiveCli.ts:+30/−3` and
`nonInteractiveCliAgentSession.ts:+27/−3` push block reasons
into the formatter via the existing warning-collection seam.
Two regression tests at `nonInteractiveCli.test.ts:1727-1773`
pin the `LoopDetected` and `MaxSessionTurns` cases (already-
existing block-shaped events) producing
`output.warnings ⊇ ["Loop detected, stopping execution"]` and
`["Maximum session turns exceeded"]`. The `it.each` shape
makes adding the third (`AgentExecutionBlocked`) trivial.

**STREAM_JSON mode** — emits a `JsonStreamEventType.ERROR`
event with `severity: 'warning'` per-block, instead of
batching at the end (correct shape for streaming consumers
who can't wait for a final blob). Test at
`nonInteractiveCli.test.ts:2086-2238` pins the
`AgentExecutionBlocked → ERROR{severity:warning} → Content
→ Finished` sequence, plus a "blocked event followed by a
result message still emits both" case which is the
load-bearing invariant for "block doesn't terminate the
session, just the in-flight tool".

Backward-compat surface: the new `warnings` field is
optional on `JsonOutput`, so existing JSON consumers that
key on `result/error/usage` see no schema break. The
`nonInteractiveCliAgentSession` event-translator changes
at `core/src/agent/event-translator.ts:+1/−1` and
`event-translator.test.ts:+2/−4` are minor fix-ups for the
same translation path.

## Risks / nits

- The `warnings: string[]` field accumulates raw block-reason
  strings. `JsonFormatter` ANSI-strips, which is correct,
  but does not deduplicate. A flow that hits the same hook
  N times would emit N copies of the same warning. Probably
  fine (an operator wants to see the count) but worth
  confirming the intent — a one-line comment at the
  collection site naming "intentionally not deduped: count
  matters for diagnostic" would help.
- `useAgentStream.ts:+7/−2` adds the same warning-emission
  path to the interactive UI hook, which means the
  warning is now also collected in interactive mode where
  the toast already shows it. If the hook then writes to
  the same `warnings` channel that interactive UIs read,
  there's a chance of double-display (toast + final
  warnings list). Worth checking that the interactive
  output formatter doesn't read `warnings` in its render
  pass.
- The 196-line test file rewrite at
  `nonInteractiveCli.test.ts` and 203-line at
  `nonInteractiveCliAgentSession.test.ts` is the bulk
  of the diff; they're additive (new `it`/`it.each`
  blocks) so the existing test surface is preserved. No
  removed assertions. Good.
- The PR's stated repro (`repro-20859` branch, references
  the original issue with `AgentExecutionBlocked silently
  swallowed in JSON mode`) — would be worth one extra
  test asserting `output.warnings` includes
  `'Blocked by hook'` (the test at `:2098` uses that exact
  string in STREAM_JSON; the JSON-mode equivalent would
  symmetrically pin the issue's named bug).

## Verdict

**`merge-after-nits`**

Real diagnostic gap closed, the schema extension is
backward-compatible (optional field), the streaming-mode
choice (per-block `ERROR{severity:warning}` instead of
end-of-run batch) is the right shape for streaming
consumers, and the `it.each` test pattern leaves a clear
extension seam for future block-event types. Nits are
"add a JSON-mode test naming the original issue's exact
string", "confirm interactive doesn't double-display",
and a one-line dedup-intent comment.

## What I learned

When a UI affordance (here: the interactive toast for
`AgentExecutionBlocked`) doesn't have a parallel in the
machine-readable output channels, the bug shape is
"silent semantic divergence between humans and scripts"
— the script consumer thinks "no error" means "nothing
went wrong" but actually means "the UI told a human and
forgot to tell you". The fix shape is symmetric: every
event that warrants UI surfacing also warrants a
machine-readable surface, with the same severity
implicit in both.
