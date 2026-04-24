# BerriAI/litellm PR #26446 — fix(caching): handle list-based responses and message key variations in qdrant semantic cache

- **Repo:** BerriAI/litellm
- **PR:** [#26446](https://github.com/BerriAI/litellm/pull/26446)
- **Head SHA:** `dd6db166beb1e401dd80db3f7a9b88cef789bf77`
- **Size:** +62/-32 across 2 files
- **Reviewer:** Bojun (drip-23)

## Summary

Two-in-one fix to `litellm/caching/qdrant_semantic_cache.py`:

1. The four `set_cache` / `get_cache` / `async_set_cache` /
   `async_get_cache` methods previously did `messages = kwargs
   ["messages"]` and then concatenated `message["content"]` directly.
   That crashed (`KeyError`) on call sites that pass `kwargs
   ["message"]` (singular) and crashed (`TypeError`) on calls where
   `content` is a list (multimodal Anthropic-style content blocks)
   rather than a string.

2. Pulls in the existing `get_str_from_messages` helper from
   `prompt_templates/common_utils` so the prompt extraction is
   consistent with how the rest of litellm flattens message content.

Companion test file gains two new test functions covering the
list-response cache hit and set paths.

## What's changed

### `litellm/caching/qdrant_semantic_cache.py` (+30/-32)

Each of the four cache entry points (sync set, sync get, async set,
async get) gets the same three-line transformation. From the diff
(lines 19–27 for the sync set; identical pattern at lines 35–43,
51–59, 67–75):

Before:
```python
messages = kwargs["messages"]
prompt = ""
for message in messages:
    prompt += message["content"]
```

After:
```python
messages = kwargs.get("messages") or kwargs.get("message")
if not messages:
    print_verbose("No messages provided for semantic caching")
    return
prompt = get_str_from_messages(messages)
```

Three semantic shifts:

- **Soft access via `.get("messages") or .get("message")`** — accepts
  both the plural and singular kwarg shapes that callers in the wild
  apparently use. The `or` chain means an empty list (`[]`) falls
  through to the singular check, which is fine for the empty-input
  case but may be slightly surprising if a caller intentionally passes
  `messages=[]` (empty conversation) — that case now hits the
  `print_verbose("No messages...") + return` early-exit branch.

- **Early `return` on missing messages** — previously `KeyError`,
  now returns `None` silently after a verbose log line. For
  `set_cache` this is fine (nothing to cache, no harm done). For
  `get_cache` it means a cache miss instead of an exception. The new
  failure mode is strictly safer than the old one: a missing-message
  call now degrades to "no cache" rather than killing the request.

- **Delegates content extraction to `get_str_from_messages`** — this
  is the load-bearing fix. The old `prompt += message["content"]`
  blows up the moment `content` is a `list[dict]` (Anthropic content
  blocks: text/image/tool_use). The shared helper handles those by
  flattening to text. Side benefit: any future content-shape change
  (e.g. when reasoning_content blocks become routine) is handled in
  one place rather than four.

### `tests/test_litellm/caching/test_qdrant_semantic_cache.py` (+102 lines)

Diff lines 91–187. Adds two tests:

- `test_qdrant_semantic_cache_set_list_response` (lines 91–134) —
  asserts that calling `set_cache` with `value=["item1", "item2"]`
  and a single-message `messages=[{"content": "..."}]` triggers
  `qdrant_cache.sync_client.put` (i.e., the upsert lands).

- `test_qdrant_semantic_cache_get_list_response_hit` (lines 137–186) —
  asserts that when the qdrant search returns a payload with
  `"response": '["item1", "item2"]'` (JSON-encoded list-as-string),
  `get_cache` returns the parsed Python list `["item1", "item2"]`.

The list-response path is the second half of the PR title and the
tests are tight on it. **However**, neither new test exercises the
new defensive-path logic from the production code change itself —
no test for `kwargs.get("message")` (singular) handling, no test for
the `if not messages: return` early-exit branch, and no test for
list-shaped `content` (multimodal blocks). All three of those are
the actual bug fixes described in the PR title; the tests
demonstrate the *value* shape (list response) but not the *key*
shape (message vs messages, list-content blocks).

## Concerns

1. **Test coverage misses the bug-fix branches**

   See above. The PR is titled "handle list-based responses **and
   message key variations**" but only the response side gets test
   coverage. Add at minimum:
   - `messages_singular_kwarg` test → call with `message=[...]`
     (singular) and assert no crash, upsert called.
   - `messages_missing_kwarg` test → call with neither `messages`
     nor `message` and assert early-exit (no `put` call, no raise).
   - `multimodal_content_block` test → call with
     `messages=[{"content": [{"type": "text", "text": "hi"}]}]` and
     assert the prompt fed to embedding is `"hi"` (or whatever
     `get_str_from_messages` produces).

2. **Silent fallback masks integration bugs**

   Changing `kwargs["messages"]` (`KeyError`) to `kwargs.get(...)
   or kwargs.get(...)` plus a verbose-only log is the right defensive
   move for production, but it also means a caller that *typos* the
   kwarg ("messags") will silently get cache-miss / no-cache forever
   with only a verbose-log breadcrumb. Worth raising a real warning
   (logger.warning, not just print_verbose) for the "neither key
   present" case so it shows up in default logging.

3. **`or` chain edge case on empty list**

   `kwargs.get("messages") or kwargs.get("message")` — if a caller
   passes `messages=[]` (legitimately empty conversation, perhaps for
   a system-prompt-only call) the `or` falls through to the singular
   key, then to the early-exit. If both kwargs are explicitly `[]`,
   we silently no-op. This is probably the right call (semantic
   caching of an empty conversation is meaningless) but it's worth
   the comment.

4. **`get_str_from_messages` import path coupling**

   Adding `from litellm.litellm_core_utils.prompt_templates.
   common_utils import get_str_from_messages` (diff lines 9–11) at
   the top of `qdrant_semantic_cache.py` couples the cache module
   to that helper's location. If a future refactor moves
   `common_utils`, this import breaks. Not a blocker, but worth
   noting that this helper is now load-bearing for cache correctness
   and should be kept stable.

5. **Behavior change is silent in CHANGELOG / release notes**

   The PR transitions four entry points from "raises on missing
   messages" to "silently no-ops on missing messages." Any downstream
   that *catches* the KeyError loses its signal. Worth a one-line
   note in the PR body / release notes acknowledging the failure-mode
   shift.

## Verdict

`merge-after-nits` —

- expand test coverage to cover the three actual bug-fix branches
  (singular kwarg, missing kwargs, list-shaped content);
- escalate the "neither key present" case from `print_verbose` to
  `logger.warning` so caller-typo bugs surface in default logging;
- one-line release note on the silent-fallback semantic shift.

The fix is correct in direction and the use of the existing helper
is the right call.

## What I learned

Semantic-cache layers are repeatedly the place where multimodal
content-block churn first surfaces — they were written when
`message["content"]` was reliably a string, and Anthropic's switch
to `list[ContentBlock]` four releases ago has been generating these
"works on text-only, crashes on multimodal" bugs across litellm,
opencode, and crush throughout 2026-W17 (see also #26285, #26426,
#24146, #24150 — all reasoning-content / content-block issues
landed during the same drip period). The general fix pattern is
the same: route every `content` access through one helper, never
inline the loop.
