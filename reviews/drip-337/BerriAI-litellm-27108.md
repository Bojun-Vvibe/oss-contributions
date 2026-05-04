# BerriAI/litellm #27108 — fix(usage-ai-chat): resolve proxy aliases before acompletion

- **Head SHA reviewed:** `3f695379d6d448fa8ee45f2e929f5269557b436a`
- **Size:** +82 / -1 across 2 files
- **Verdict:** merge-after-nits

## Summary

The "Ask AI" usage chat endpoint was passing the user-facing model name
(potentially a proxy alias) directly into `litellm.acompletion`,
bypassing the alias-resolution path that the rest of the proxy honors.
This PR adds `_resolve_usage_chat_model_alias()` and threads its result
through `stream_usage_ai_chat`. Closes #27046.

## What I checked

- `litellm/proxy/management_endpoints/usage_endpoints/ai_usage_chat.py:429-449`
  — the new helper checks two alias surfaces:
  1. `litellm.model_alias_map` (global map)
  2. `llm_router.model_group_alias` (per-router map)

  Order is correct (global wins over router), and both lookups are
  guarded by `is not None` / `getattr(..., {})`. Good defensive shape.
- `ai_usage_chat.py:441-447` wraps the router import in a bare
  `except Exception: pass`. This is overly broad — at minimum log at
  debug level so silent fall-throughs don't mask import errors during
  development. Suggest:
  ```python
  except Exception as e:
      verbose_logger.debug(f"router alias resolution skipped: {e}")
  ```
- `ai_usage_chat.py:558-560` — `selected_model` keeps the original
  user input; `resolved_model` is what gets passed to `acompletion`.
  Names are clear. The `DEFAULT_COMPETITOR_DISCOVERY_MODEL` fallback
  applies *before* alias resolution, which is correct (default is
  itself a real model name, not an alias).
- `tests/.../test_ai_usage_chat.py:180-238` — two new test cases
  cover both alias surfaces. The router test correctly uses
  `patch.dict("sys.modules", ...)` to avoid importing the real
  proxy_server. The assertion `first_call_kwargs["model"] ==
  "anthropic/claude-3-5-sonnet"` directly proves the resolution
  happened before the call. Solid.
- One missing case: what if both `model_alias_map` *and*
  `model_group_alias` define the same key? Current code returns the
  global one. Worth a third test to pin precedence so a future
  refactor doesn't silently flip it.
- The `resolved_model` variable name shadows itself within the helper
  (input arg `model` → `resolved_model = model.strip()` → reassigned).
  Minor; consider `candidate = model.strip()` then return.

## Risk

Low. Tightly scoped to the Ask-AI usage chat path. The fallback
returns the input unchanged, so worst case is a no-op.

## Recommendation

Merge after:

1. Replace the bare `except Exception: pass` with a debug log so
   import-time issues are observable.
2. Add a precedence test (alias defined in both maps → global wins).
3. PR body notes test fails locally on missing `tokenizers` — the CI
   matrix should still cover it; confirm green there before merge.
