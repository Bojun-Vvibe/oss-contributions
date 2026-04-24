# BerriAI/litellm PR #26453 — Fix qdrant semantic cache

- **Repo:** BerriAI/litellm
- **PR:** [#26453](https://github.com/BerriAI/litellm/pull/26453)
- **Head SHA:** `607596cc74a4742678c2e50b8886c1a881f711a1`
- **Author:** borisduin (Boris Antonio Duin)
- **Size:** +117/-18 across 2 files
- **Reviewer:** Bojun (drip-24)

## Summary

Fixes three bugs in `QdrantSemanticCache` that compound:

1. `value = str(value)` for serialization produces Python-repr
   strings (single quotes, `'item1'`) that JSON parsers reject —
   forcing downstream code into fragile `ast.literal_eval`
   fallbacks. Fix: `value = json.dumps(value)`.
2. Prompt extraction blindly concatenated `message["content"]`
   for every message in `messages`, which crashes with
   `TypeError: can only concatenate str (not "list") to str`
   the moment a multimodal content block lands. Fix: route
   through `get_str_from_messages(messages)` helper which knows
   how to flatten content blocks.
3. Some callers pass the prompt under a singular `'message'` key
   instead of plural `'messages'`. Fix:
   `kwargs.get("messages") or kwargs.get("message")`.

Plus an early-return guard when neither key is set (with
`print_verbose` instead of raising).

## Key changes

### `litellm/caching/qdrant_semantic_cache.py` (+~25/-15)

Same four-line pattern is applied to all four cache methods
(`set_cache`, `get_cache`, `async_set_cache`, `async_get_cache`).
The `set_cache` diff at line 178:

```python
- messages = kwargs["messages"]
- prompt = ""
- for message in messages:
-     prompt += message["content"]
+ messages = kwargs.get("messages") or kwargs.get("message")
+ if not messages:
+     print_verbose("No messages provided for semantic caching")
+     return
+ prompt = get_str_from_messages(messages)
```

And in both `set_cache` paths (line 200, async at line 332):

```python
- value = str(value)
+ value = json.dumps(value)
  assert isinstance(value, str)
```

The `json.dumps` change is the one I most care about. With
`str(value)`:
- `value = ["a", "b"]` becomes `"['a', 'b']"` — invalid JSON.
- `value = {"k": "v"}` becomes `"{'k': 'v'}"` — invalid JSON.
- Retrieval downstream has to `ast.literal_eval` to recover.

With `json.dumps(value)`:
- `value = ["a", "b"]` becomes `'["a", "b"]'` — valid JSON.
- Retrieval is a clean `json.loads`.

The retrieval test `test_qdrant_semantic_cache_get_list_response_hit`
at line ~470 sets `"response": '["item1", "item2"]'` in the mocked
qdrant payload and asserts `result == ["item1", "item2"]` — pinning
the full round-trip.

### Test coverage (`tests/test_litellm/caching/test_qdrant_semantic_cache.py`, +95)

Two new tests:

1. `test_qdrant_semantic_cache_set_list_response` — verifies
   list responses can be cached without raising. Uses a mocked
   `_get_httpx_client` and asserts `qdrant_cache.sync_client.put`
   was called.
2. `test_qdrant_semantic_cache_get_list_response_hit` — verifies
   list responses round-trip correctly. The payload includes a
   pre-serialized `'["item1", "item2"]'` string and the test
   asserts `result == ["item1", "item2"]`.

Both tests use `MagicMock` for the http client and patch
`litellm.embedding` to return a fake vector. Reasonable
isolation.

## Concerns

1. **The four near-identical patches should be a helper.**

   The `messages = kwargs.get("messages") or kwargs.get("message")
   / if not messages: return / prompt = get_str_from_messages(...)`
   block is copy-pasted across four methods. A
   `_extract_prompt_from_kwargs(kwargs) -> str | None` helper
   would dedup ~20 lines and ensure all four methods stay in sync
   on a future fix (e.g. when someone adds a third key spelling
   like `"prompt"`).

2. **Silent return on missing messages.**

   `print_verbose("No messages provided ...")` + bare `return`
   means a caller who forgot to pass messages gets a silent
   cache miss / no-op cache write, with the only signal being a
   verbose log line. For the cache miss case (`get_cache`
   returning `None`) that's fine — semantically equivalent to a
   cache miss. For the cache write case (`set_cache` returning
   without writing), the caller may believe the value was
   cached. A `verbose_logger.warning(...)` instead of
   `print_verbose(...)` would surface this at the default log
   level.

3. **`json.dumps(value)` on `value` of unknown type.**

   `value` is the response object being cached. If it's a
   `litellm.ModelResponse` (a Pydantic model), `json.dumps()`
   will raise `TypeError: Object of type ModelResponse is not
   JSON serializable`. The old `str(value)` would have given
   the Pydantic repr (lossy but non-throwing). The PR doesn't
   add a `default=` handler. This is *probably* fine because
   `litellm` typically converts to dict before caching, but
   worth a `default=lambda o: o.model_dump() if hasattr(o,
   "model_dump") else str(o)` for safety.

4. **No regression test for the original `TypeError` failure
   mode.**

   The PR description quotes the original error
   (`can only concatenate str (not "list") to str`) but the new
   tests don't construct a multimodal-content-block message and
   assert it doesn't raise. A test like:

   ```python
   messages = [{"content": [{"type": "text", "text": "hi"},
                            {"type": "image_url", ...}]}]
   ```

   would directly pin the regression. Right now the new
   `_list_response` tests cover the `str(value)`→`json.dumps`
   fix but not the `prompt += message["content"]` fix.

5. **`get_str_from_messages` import path.**

   The new import is from
   `litellm.litellm_core_utils.prompt_templates.common_utils`.
   That module path is deep — worth verifying it's a stable
   public-ish API rather than an internal helper that could move.

## Verdict

`merge-after-nits` — three real bugs fixed, the JSON-vs-repr
serialization fix in particular is overdue and removes a class
of `ast.literal_eval` workarounds. The new tests cover the
list-response round-trip cleanly. Five small follow-ups:

- factor the four-method copy-paste into a helper,
- add a multimodal-content-block test that pins the original
  `TypeError`,
- handle Pydantic models in `json.dumps` via `default=`,
- bump the silent-return log level above `print_verbose`,
- confirm `get_str_from_messages` is a stable import path.

## What I learned

`str(obj)` for serialization is a long-standing Python footgun
that survives quietly in cache layers because the failure mode
isn't a crash — it's a silent
"the cache works on writes but every read goes through
`ast.literal_eval` because what we wrote isn't real JSON". The
fix is always `json.dumps`, but the migration cost (existing
cache entries are in repr format and can't be parsed by
`json.loads`) is non-trivial for production deployments — would
want a one-shot migration that re-encodes existing entries, or
a dual-decode path (try `json.loads`, fall back to
`ast.literal_eval` once during a rollover window).
