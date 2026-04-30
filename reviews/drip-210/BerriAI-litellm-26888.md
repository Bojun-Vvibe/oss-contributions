# BerriAI/litellm#26888 — Fix Responses API developer/system message merging

- PR: https://github.com/BerriAI/litellm/pull/26888
- Head SHA: `aeb46cbc6c6b7e59e7fc859f41a1ec9ef57572f8`
- Author: Genmin
- Files: 2 changed (`litellm/llms/base_llm/base_utils.py` +66/−8, new test
  file `tests/test_litellm/litellm_core_utils/test_base_llm_base_utils.py` +115)
- Closes #26879

## Context

`map_developer_role_to_system_role` in
`litellm/llms/base_llm/base_utils.py` translates OpenAI-style `developer`
role messages into `system` for non-OpenAI providers. The previous
implementation was a one-pass loop that replaced every `developer` with a
fresh `{role: system, content: ...}` dict, which meant the Responses API
bridge — which feeds `instructions` (as `system`) **plus** the user's
`developer`-role input array — produced *two* leading system messages on
the way to (e.g.) Anthropic. Anthropic and several other backends silently
keep only the first, so the second `system` block was being dropped.

## Design

The new pass (`base_utils.py:215-282`) splits the message list into a
*leading system-equivalent run* (consecutive `developer`/`system` from
index 0) and a *tail* (everything after the first non-system role).

**Early return for the common case** (`:218-219`):
```python
if not any(m["role"] == "developer" for m in messages):
    return messages
```
Returns the input list `is`-identically — the test at
`test_base_llm_base_utils.py:8-15` asserts exactly `result is messages`,
which is the right invariant: no allocation, no copy, no deep-copy of large
content blocks for the 99% case where there's no `developer` role at all.

**Leading-run merge** (`:222-238`). The while loop walks while
`messages[idx]["role"] in {"developer", "system"}`. The first such message
seeds `leading_system_message` (with `role` forced to `"system"`,
preserving any other top-level keys via `dict(m)`), and each subsequent
content is collected into `leading_system_contents`. After the loop,
`_merge_system_message_contents` collapses the list into a single
`content`.

**Tail conversion** (`:240-249`). For the rest of the messages, any
`developer` is rewritten in place as a fresh dict with `role: "system"`;
any other role is appended unchanged. This preserves the *position* of a
later `developer` message instead of hoisting it into the leading block —
the test at `:74-86` pins this behavior (a `developer` after a `user` stays
in its original position, just retyped).

**Content merging** (`:259-273`). `_merge_system_message_contents`
specializes on whether all contents are strings/None or include structured
blocks. String path: `"\n\n".join(content for content in contents if
content)` — empty/None entries are filtered, no leading/trailing `\n\n`.
Block path: blocks are concatenated with a `{"type": "text", "text":
"\n\n"}` separator block between non-empty groups (`:269-272`). The test at
`:34-50` covers the structured case including a `{role: developer,
content: ""}` and a `{role: developer, content: None}` interleaved with a
real block — both correctly skipped.

## What's correct

- **`is`-identity preservation in the no-op case** is the kind of thing
  that only matters until a downstream caller relies on it (e.g., a
  metrics decorator that hashes the input list by `id()`), and at that
  point it matters a lot. Pinning it in a test was the right move.
- **The leading-run boundary is `messages[idx]["role"] in {"developer",
  "system"}`** — not "first system message" or "all system messages".
  This is the only definition that produces a single merged leading
  system message *and* leaves a `developer` message after a `user` alone.
- **The Responses-bridge integration test** at `:99-115` exercises the
  full path: `LiteLLMCompletionResponsesConfig.transform_responses_api_request_to_chat_completion_request`
  produces an `instructions: "Follow..."` + `developer: "Prefer..."`
  layout, and the merge collapses them into one `system: "Follow...\n\nPrefer..."`.

## Risks / nits

- **Mutation safety**: `leading_system_message = dict(m)` is a shallow copy.
  If `m["content"]` is a list of dicts and the merge path appends a
  separator block to it (via `merged_blocks.extend(content_blocks)` at
  `:271`), the original `m["content"]` list is *not* mutated because
  `merged_blocks` is a fresh list. Confirmed by re-reading `:267-272`.
  Worth a one-line comment explaining "we never mutate `messages[i]`".
- **Empty result**: if every leading entry has `content=""` or `None`,
  `_merge_system_message_contents` returns `""` (string path) or `[]`
  (block path), and the leading block is still emitted with that empty
  content. Most providers tolerate this, but a `if leading_system_message
  and content:` guard before `new_messages.append(...)` would be tidier.
  Not a correctness issue.
- The `cast(AllMessageValues, ...)` calls at `:236` and `:246` are
  necessary because `dict(m)` widens the TypedDict — the cast is the
  conventional pyright/mypy escape hatch and is fine.

## Verdict

**`merge-as-is`**

Bug shape (silently-dropped second system message) is real and provider-
specific (Anthropic). Fix shape (split leading run, merge contents,
preserve tail position) is the right one — not "drop developer everywhere"
which would lose ordering semantics. Tests cover the happy path,
structured-content path, tail-position preservation, the absence-of-leading-
system case, and the Responses-bridge integration. The `is`-identity early
return is a nice efficiency win pinned by an explicit assertion.

## What I learned

When a translation function is called on the hot path, the no-op identity
case deserves an explicit `is messages` test — without it, a future
"defensive copy" refactor will silently double allocations across every
request. The pattern of *split leading homogeneous run, merge it, leave the
tail alone* generalizes: it's the right shape any time a sequence has a
"prelude" with merge semantics distinct from the body. The structured-
content separator-block trick (`{"type": "text", "text": "\n\n"}`) is
worth remembering — it composes through any provider that already handles
block lists, with no special-casing in the provider adapter.
