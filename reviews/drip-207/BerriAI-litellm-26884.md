# BerriAI/litellm #26884 — fix: merge responses developer messages into system prompt

- Head SHA: `2cc9940ea3c5ffeba552255d87c47e6acfeae6dc`
- Files: `litellm/responses/litellm_completion_transformation/transformation.py`, `tests/test_litellm/responses/litellm_completion_transformation/test_litellm_completion_responses.py`
- Size: +111 / -1

## What it does

Fixes #26879. The Responses API → Chat Completions transformation in
`LiteLLMCompletionResponsesConfig.transform_responses_api_input_to_messages`
was emitting a separate `system` message *per* `developer`/`system`
input item plus a separate one for the top-level `instructions`
field. Many providers either reject multiple system messages or
silently keep only the first/last, so behavior was provider-dependent
and opaque.

The PR adds two helpers and a single call-site change at
`transformation.py:285`:

- `_get_text_from_message_content` (`transformation.py:286-303`)
  normalizes a Responses-API `content` value into a flat string. It
  handles `None`, plain `str`, and `list[str | dict]` shapes, and
  inside the dict shape it accepts both `type: "text"` and
  `type: "input_text"` (which is the variant the Responses API
  actually emits). Unknown content types are silently dropped — only
  text parts contribute to the merged system prompt.
- `_merge_responses_system_messages`
  (`transformation.py:305-360`) walks the message list, splits it into
  `system_parts: List[str]` (collected from any message whose role is
  `"system"` or `"developer"`) and `non_system_messages` (everything
  else, in original order). If `system_parts` is non-empty, it returns
  `[ChatCompletionSystemMessage(role="system", content="\n\n".join(system_parts))] + non_system_messages`;
  if empty (no system/developer at all) it returns `messages`
  unchanged, which preserves the no-op path.

The transformation now returns the merged result instead of the raw
list at `transformation.py:285`.

The test at
`test_litellm_completion_responses.py:811-846` constructs the exact
issue-#26879 shape: top-level `instructions: "You are helpful."`, two
`developer`-role inputs (one plain-string, one
`[{"type": "input_text", "text": "..."}]`), and a user message. It
asserts the result is exactly two messages — one system with content
`"You are helpful.\n\nUse concise responses.\n\nNever expose secrets."`
and the user message — pinning down both the merge order
(instructions-first, then developer messages in input order) and the
join separator.

## What works

Splitting `_get_text_from_message_content` out as a separate static
method is the right structural call: the same content-shape
flattening logic is needed in at least three other places in this
file (anywhere a Responses-API content blob meets a
chat-completions-shaped consumer), and now there's one canonical
implementation to call.

The `role in {"system", "developer"}` membership check is correct —
the Responses API does emit `developer` as a distinct role per the
OpenAI spec, and treating both as "this is system-prompt material"
matches what every Chat Completions provider expects to receive.

The early-return when `system_parts` is empty
(`transformation.py:354-355`) preserves the exact pre-PR message
list rather than constructing a new list — small thing, but it means
the no-op path doesn't allocate or change message identity, so any
downstream code that's doing `is`-based caching keeps working.

The test uses both content shapes (plain string and `input_text` part)
in a single assertion, which is the right move because the dual
handling in `_get_text_from_message_content` is the part most likely
to silently regress.

## Concerns

1. **Order coupling between `instructions` and developer messages is
   load-bearing but undocumented.** The test asserts
   `"You are helpful.\n\nUse concise responses.\n\nNever expose secrets."`,
   which only holds because the upstream
   `transform_responses_api_input_to_messages` happens to prepend the
   `instructions` system message before walking the input list. If
   anyone reorders that upstream loop, the test will keep passing
   (because instructions is still in `system_parts`) but the merged
   ordering will silently change. A comment in
   `_merge_responses_system_messages` explaining "merge order is
   input order, callers responsible for instructions-first
   positioning" would harden against this.
2. **`assistant`-role content with a `system`-typed nested part is
   not handled.** The walker only checks the outer `role`, so a
   message like `{"role": "user", "content": [{"type": "system",
   "text": "..."}]}` (which is not currently emitted by the Responses
   API but is technically representable) would not get merged.
   Probably fine — the Responses API doesn't emit that — but worth a
   one-line comment.
3. **`isinstance(message, dict)` guard at `transformation.py:336`
   silently drops typed-message inputs.** The function signature
   accepts five message types, four of which are TypedDict-derived
   (so `isinstance(msg, dict)` is true), but `Message` from the
   pydantic-models path is *not* a dict. If a `Message` instance is
   passed in with `role="developer"`, the `role = ... else None`
   branch at `:336` returns `None`, the `role in {...}` check fails,
   and the message lands in `non_system_messages` as a raw `Message`
   object — which is the pre-PR behavior, so not a regression, but
   the new code creates the impression that all five input types are
   handled when really only the four dict-shaped ones are. Either
   tighten the type signature to `List[dict]` here or branch on
   `getattr(message, "role", None)` for the pydantic path.
4. **Empty-content developer message is silently dropped.** If a
   developer message has `content: ""` or `content: None`,
   `_get_text_from_message_content` returns `""`, the
   `if text_content:` guard at `:344` skips it, and the message is
   dropped from both `system_parts` and `non_system_messages`. That's
   probably the desired behavior (an empty system prompt has no
   value), but it's a silent message loss — worth a debug log.

Verdict: merge-after-nits

## What I learned

The "split into kinds, then merge one kind, then concatenate"
pattern is the right shape for prompt-shape transformations because
it preserves the relative order of the *other* kind for free. The
naive alternative — walk the list, build a new list, conditionally
append-or-merge as you go — gets the same result for this input but
is harder to reason about when an empty-content edge case shows up:
in the naive form, the empty developer message either creates a
zero-content system message or you have to add a guard at the
append site; in the split-then-merge form, the guard lives in
exactly one place (the `if text_content:` check) and you can read it
in isolation. Worth remembering when designing the next prompt
transformation.
