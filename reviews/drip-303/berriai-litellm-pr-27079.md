# BerriAI/litellm #27079 — fix(gemini): generate unique tool_call_ids in GoogleGenAI adapter

- URL: https://github.com/BerriAI/litellm/pull/27079
- Head SHA: `391b9c519f3c5b1b949d1d156ad510a737a888eb`
- Scope: +537 / -105 across 3 files
- Closes: #27078

## Summary

`GoogleGenAIAdapter._transform_contents_to_messages()` was generating tool
call IDs as `f"call_{func_name}"`. When a model turn contained two
`functionCall` parts for the same function (e.g. `get_weather` for two
cities), both got the same id `call_get_weather`. Same for the matching
`functionResponse` parts. Downstream providers (anything OpenAI-shaped)
cannot distinguish which response belongs to which call, breaking parallel
tool calls of the same function.

The fix replaces deterministic IDs with `f"call_{uuid.uuid4().hex[:24]}"`
and maintains a per-function-name FIFO queue so each `functionResponse`
matches the correct preceding `functionCall`.

## What I checked

- `litellm/google_genai/adapters/transformation.py:381+` — new
  `pending_tool_call_ids: Dict[str, List[str]]` lives at the top of the
  loop over `contents`, scoped per-conversation. Correct lifetime: each
  call to `_transform_contents_to_messages` starts fresh, so IDs don't
  leak across conversations.
- The two helper extractions (`_transform_user_parts`,
  `_transform_model_parts`) reduce nesting and are pure refactor; they
  receive `pending_tool_call_ids` by reference so model-side appends are
  visible to subsequent user-side pops. Correct.
- FIFO matching: `pending_ids.pop(0)` is O(n) but n is small (number of
  unanswered calls for a single function name in a single turn). Fine.
- Orphan handling: if a `functionResponse` arrives with no preceding
  `functionCall` for that name, the code falls back to a fresh uuid
  rather than raising. This matches real-world traces (e.g. resumed
  conversations where the call lives in older history) — better to
  produce a syntactically valid message than crash. The
  `test_orphan_function_response_gets_fresh_id` case covers this.
- `uuid.uuid4().hex[:24]` — 24 hex chars = 96 bits of entropy. Plenty
  for collision avoidance, and matches OpenAI's `call_` id length
  conventions reasonably closely.
- Tests (`tests/test_litellm/google_genai/test_google_genai_adapter_tool_call_id.py`,
  +404, 9 cases) cover: single call uniqueness, duplicate names get
  distinct IDs, response→call FIFO matching, mixed function matching,
  multi-turn flows, orphan responses, content serialization. Solid matrix.

## Nits

1. **`uuid` import** is added at module top (`+import uuid`) — good.
2. **Truncation to 24 chars.** Using `[:24]` is fine for entropy but loses
   information vs the full 32-char hex. If the goal is to mirror OpenAI's
   id format (`call_<24-chars>`), a brief comment saying so would help
   future readers. Otherwise consider `uuid.uuid4().hex` to avoid the
   magic number.
3. **FIFO order assumption.** The matching is FIFO by function name. Gemini
   `functionResponse` parts arrive in the same order the model emitted them
   (per Google's spec), but if a future API change reorders them, FIFO
   matching would silently mismatch. Worth a short docstring on
   `_transform_user_parts` noting the FIFO assumption and where it comes
   from.
4. **`pending_tool_call_ids` is unbounded** within a single conversation.
   For pathological cases (1000s of calls with no responses), it grows
   without bound. Not a real concern in practice — Gemini's per-turn
   structure tightly bounds this — but a one-line comment would document
   the invariant.
5. The PR also touches `test_google_genai_adapter.py` with a single-line
   change (`+1/-1`). Worth confirming in the commit message that this is
   adjusting an existing test for the new id format, not a drive-by.

## Risk

Low. Bug fix in a clearly-broken path (same-id collisions are
unambiguously wrong). New behavior is strictly more correct. Test coverage
is strong. The refactor (extracting `_transform_user_parts` /
`_transform_model_parts`) is bundled but improves readability.

## Verdict

`merge-after-nits` — primarily the FIFO docstring and the magic-number
comment on `[:24]`.
