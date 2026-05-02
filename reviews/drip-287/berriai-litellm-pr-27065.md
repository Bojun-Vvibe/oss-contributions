# BerriAI/litellm PR #27065 — feat(anthropic): surface compaction usage iterations data

- Head SHA: `6beb5c8ea10e3a35d161a65cef788c4379e793d0`
- URL: https://github.com/BerriAI/litellm/pull/27065
- Size: +116 / -2, 3 files
- Verdict: **request-changes**

## What changes

Anthropic's API is now returning a `usage.iterations[]` array on
responses where compaction occurs (Issue #27060). This PR surfaces
that data:

1. `litellm/llms/anthropic/chat/transformation.py:1716-1720` reads
   `usage_object.get("iterations")`. If present, **overwrites**
   `prompt_tokens`/`completion_tokens` with the sum across iterations.
2. `litellm/llms/anthropic/chat/transformation.py:1774` derives
   `raw_input_tokens = prompt_tokens - cache_creation_input_tokens - cache_read_input_tokens`
   instead of reading `usage_object["input_tokens"]` directly.
3. `litellm/types/utils.py:1543` adds `iterations: Optional[List[Any]]`
   to `Usage` and threads it through `__init__`.
4. New `tests/test_anthropic_compaction_usage.py` covers the basic
   summation and a cache-aware variant.

## What looks good

- Capturing per-iteration usage is the right primitive — billing
  reconciliation against compaction is currently impossible with the
  pre-aggregated top-level fields.
- Adding `iterations` to the typed `Usage` model rather than smuggling
  it through `**params` is the right call.

## Issues that need addressing before merge

### 1. `prompt_tokens`/`completion_tokens` are uninitialized on the iteration path

In `calculate_usage` the new block (lines 1716–1720):

```python
iterations: Optional[List[Any]] = usage_object.get("iterations")
if iterations:
    prompt_tokens = sum(it.get("input_tokens", 0) or 0 for it in iterations)
    completion_tokens = sum(it.get("output_tokens", 0) or 0 for it in iterations)
```

`prompt_tokens` and `completion_tokens` are introduced as new locals here
but the **else branch** (no iterations) does not assign them. Later code at
line 1774 unconditionally references `prompt_tokens` to compute
`raw_input_tokens`. This will raise `UnboundLocalError` on the
overwhelmingly common path where Anthropic does not return
`iterations` (every non-compaction response). This is a hard regression
across all Anthropic traffic and must be fixed before merge.

Fix: initialize both above the `if iterations:` branch, e.g.
```python
prompt_tokens = usage_object.get("input_tokens", 0) or 0
completion_tokens = usage_object.get("output_tokens", 0) or 0
if iterations:
    prompt_tokens = sum(...)
    completion_tokens = sum(...)
```

### 2. Suspicious unrelated rename in `types/utils.py:1685`

The diff renames `self._cache_read_input_tokens = params["prompt_cache_hit_tokens"]`
to `self._prompt_cache_hit_tokens = ...`. This is **not** related to
compaction iterations and changes which attribute downstream code reads
for DeepSeek-style cache-hit reporting. Either:
- Split this into its own PR with a proper rationale, or
- If it's intentional, document why the previous attribute name was
  wrong and confirm no consumer reads `_cache_read_input_tokens`.

As-is this looks like a search-and-replace casualty and is the kind of
change that breaks billing in production a week later.

### 3. Test coverage misses the regression in #1

`test_anthropic_compaction_usage_calculation` only exercises the
`iterations`-present path. Add a test asserting that a vanilla
`{"input_tokens": N, "output_tokens": M}` (no iterations) still
returns `usage.prompt_tokens == N` and `usage.completion_tokens == M`.

## Nits

- `iterations: Optional[List[Any]]` — `Any` is permissive on purpose,
  but the test already uses `it["type"]`. Worth introducing a typed
  `AnthropicUsageIteration(TypedDict)` so consumers don't all reinvent
  the shape.

## Risk

**High** if merged as-is — see issue #1. Once that's fixed, low.
