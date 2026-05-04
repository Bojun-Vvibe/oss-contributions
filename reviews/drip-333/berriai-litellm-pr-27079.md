# BerriAI/litellm PR #27079 — fix(gemini): generate unique tool_call_ids in GoogleGenAI adapter

- **Link:** https://github.com/BerriAI/litellm/pull/27079
- **Head SHA:** `9cf922b096883a2db95be0770379edc694b8713b`
- **Verdict:** `merge-after-nits`

## What it does

Refactors `GoogleGenAIAdapter._transform_contents_to_messages` (in
`litellm/google_genai/adapters/transformation.py`) to (a) generate unique
uuid-based `tool_call_id` values for every `functionCall` part instead of
the prior collision-prone `f"call_{name}"`, and (b) match each
`functionResponse` to the correct preceding `functionCall` by a per-name
FIFO queue (`pending_tool_call_ids: Dict[str, List[str]]`).

Key sites:

- `transformation.py:18` — `import uuid`.
- `transformation.py:380-388` — `pending_tool_call_ids` keyed by function
  name, threaded into both halves.
- New `_transform_user_parts` at `transformation.py:391-417` — the
  `functionResponse` branch at lines 124-138 pops the matching id; falls
  back to `f"call_{uuid.uuid4().hex[:24]}"` when no preceding call exists
  (defensive — should not happen with well-formed Gemini contents).
- New `_transform_model_parts` at `transformation.py:430-459` — the
  `functionCall` branch at lines 222-226 generates `f"call_{uuid.uuid4().hex[:24]}"`
  and registers it in `pending_tool_call_ids.setdefault(name, []).append(id)`
  for the next user turn to consume.

## Design analysis

- **Per-name FIFO matching is correct for Gemini's wire format.** Gemini
  emits `functionResponse` parts in the same order as the preceding
  `functionCall` parts that produced them; FIFO-by-name is the canonical
  matching primitive when the adapter has to invent ids that the wire
  format doesn't supply. Without name-bucketing, a turn that has
  `[get_weather(London), get_time(UTC), get_weather(Paris)]` followed by
  responses `[weather_resp_London, time_resp_UTC, weather_resp_Paris]`
  would still match correctly because each name has its own FIFO queue.
- **24-hex-char ID** at `transformation.py:225` is safely below the 64-char
  limit the OpenAI tool_call_id field accepts, and `uuid4().hex[:24]` is
  ~96 bits of entropy — plenty for in-conversation uniqueness, and the
  `call_` prefix keeps the format recognizable to consumers that pattern-
  match on it.
- **Function decomposition is the right shape.** The original 130-line
  inline branch in `_transform_contents_to_messages` is split into
  `_transform_user_parts` and `_transform_model_parts` so the threading
  of `pending_tool_call_ids` is explicit through method signatures rather
  than implicit closure capture. Easier to test in isolation; the new
  test file `test_google_genai_adapter_tool_call_id.py` exercises both
  helpers directly.

## Risks / nits

1. The fallback at `transformation.py:131-132` (`matched_id = f"call_{uuid.uuid4().hex[:24]}"`)
   silently masks a malformed-content case where a `functionResponse`
   arrives without any preceding `functionCall`. That should at minimum
   `verbose_logger.warning` so misconfigured upstream callers can debug
   the issue. Today they'd only notice when the downstream LLM gets
   confused by an orphan tool message.
2. The existing test at `test_google_genai_adapter.py:309` was *weakened*
   from `"call_get_weather" in tool_msg["tool_call_id"]` to
   `tool_msg["tool_call_id"].startswith("call_")` — that's correct given
   the design change but loses the property "the id has a name component
   for debuggability." Worth recording the call's `name` in a structured
   side-table or reflecting it in the id (e.g.
   `f"call_{name[:8]}_{uuid.uuid4().hex[:16]}"`) for log-trace usability.
3. `pending_tool_call_ids` is keyed by function name only — if the same
   function is called with the *same* args twice in a row, FIFO works. If
   Gemini ever emits `functionResponse` parts out of `functionCall`
   order within a single user turn (currently it does not, but the spec
   doesn't strictly forbid it), FIFO would mismatch. Worth a docstring
   citing the Gemini spec section that pins ordering.
4. No test for the empty-args case (`functionCall.args = {}`) being
   serialized to `"{}"` rather than `"null"` — the existing
   `json.dumps(func_call.get("args", {}))` at line 233 does the right
   thing but a one-line assertion would lock it.
5. `_transform_user_parts` and `_transform_model_parts` mutate the
   shared `messages` list rather than returning a list to extend.
   Performance-equivalent; style preference is in-place mutation here
   given the function signatures already pass `messages: List[...]`,
   but a return-value style would be more functional-test-friendly.

## What I learned

The id-collision-via-name pattern is a recurring bug class for any
adapter mediating between a wire format that does not assign call ids
(Gemini, Anthropic legacy) and a wire format that requires them
(OpenAI tools). The right fix is *always* to stop encoding semantic
information into the id and instead carry a separate (id → name) lookup
where the matching happens. This PR does that with `pending_tool_call_ids`,
which is structurally the right primitive even if the FIFO-by-name
implementation has the orphan-response edge case to watch.
