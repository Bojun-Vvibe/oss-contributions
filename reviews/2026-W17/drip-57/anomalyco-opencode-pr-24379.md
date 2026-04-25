# anomalyco/opencode #24379 — fix(session): use transcript position instead of lexical ID compare in prompt loop

- URL: https://github.com/anomalyco/opencode/pull/24379
- Head SHA: `e8131301887b04668d72dfe5178cdc247f71f7bd`
- Verdict: **merge-as-is**

## What it does

`SessionPrompt.run` was deciding transcript ordering by comparing opaque
message IDs (`lastUser.id < lastAssistant.id`). The HTTP API explicitly
allows callers to supply their own `messageID` on prompt input, so a
caller-supplied ID like `msg_zzzzzzzz...` lex-sorts greater than any
`MessageID.ascending()` output and reverses the comparison. Symptom: loop
keeps running after `finish: "stop"`, the next request ends with an
assistant turn, and Anthropic via OpenRouter rejects with "last message
must be a user message."

## Diff notes

`packages/opencode/src/session/prompt.ts:1324-1344` now records array
positions during the same backward walk that already collects
`lastUser` / `lastAssistant` / `lastFinished`:

```
let lastUserIndex = -1
let lastAssistantIndex = -1
let lastFinishedIndex = -1
```

The loop-exit check at line 1369 changes from `lastUser.id <
lastAssistant.id` to `lastUserIndex < lastAssistantIndex`. The
post-finish reminder loop at 1471-1476 changes from filtering
`m.info.id <= lastFinished.id` to iterating `i = lastFinishedIndex + 1
... msgs.length`. Both substitutions are sound because `msgs` already
comes from `MessageV2.filterCompactedEffect` → `page()` which orders by
`time_created, id` — i.e. canonical transcript order.

## Risk surface

- No new traversals; same backward walk just records two extra integers.
- Behaviorally identical for OpenCode-generated monotonic IDs (where
  array index and lexical ID order agree).
- Test (`prompt.test.ts:343-407`) seeds a custom `msg_zzz...` user ID
  plus an auto-ID assistant with strictly greater `time.created` and
  asserts `prompt.loop` returns the assistant message with `0` LLM
  calls. Fails on `dev`, passes with the fix — exactly the right
  regression shape.

## Why this verdict

Tight, scoped fix for a real footgun introduced by accepting
caller-supplied IDs. Transcript order should never have been derived
from opaque-ID comparison; this corrects it without expanding scope.
Lands clean.
