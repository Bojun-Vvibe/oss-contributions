# BerriAI/litellm PR #27065 — feat(anthropic): surface compaction usage iterations data

- **Repo:** BerriAI/litellm
- **PR:** #27065
- **Head SHA:** `6beb5c8ea10e3a35d161a65cef788c4379e793d0`
- **Author:** Dotify71
- **Title:** feat(anthropic): surface compaction usage iterations data
- **Diff size:** +116 / -2 across 3 files
- **Drip:** drip-293

## Files changed

- `litellm/llms/anthropic/chat/transformation.py` (+~7) — at line ~1716, reads `iterations: Optional[List[Any]] = usage_object.get("iterations")` and, when present, reassigns `prompt_tokens = sum(it.get("input_tokens", 0) or 0 for it in iterations)` and same for `completion_tokens`. At line ~1774 changes `raw_input_tokens = usage_object.get("input_tokens", 0) or 0` to `raw_input_tokens = prompt_tokens - cache_creation_input_tokens - cache_read_input_tokens`. Threads `iterations=iterations` into the returned `Usage(...)`.
- `litellm/types/utils.py` (+~6) — adds `iterations: Optional[List[Any]] = None` field to `Usage`, accepts `iterations` kwarg in `__init__`, sets it via `if iterations is not None: self.iterations = iterations`. Also (suspicious) changes `self._cache_read_input_tokens = params["prompt_cache_hit_tokens"]` to `self._prompt_cache_hit_tokens = params["prompt_cache_hit_tokens"]`.
- `tests/test_anthropic_compaction_usage.py` (+101, new) — two parametrized-flavored functions covering the basic iteration sum and the cache-aware sum.

## Specific observations

- `transformation.py:1716-1719` — silently overwriting `prompt_tokens`/`completion_tokens` (which were just initialized via `_get_token_counts(...)` upstream of this hunk) only when `iterations` is truthy is correct for the compaction path, but the surrounding code in `calculate_usage` is dense and this rewrite happens 60+ lines before the values are first used. Add a one-line comment ("Anthropic compaction: top-level input/output_tokens exclude per-iteration usage; sum the iterations array to get the true total — Issue #27060") so the next maintainer doesn't accidentally undo it.
- `transformation.py:1774` — `raw_input_tokens = prompt_tokens - cache_creation_input_tokens - cache_read_input_tokens` is **a behavior change for the non-iterations path too**. Previously this was `usage_object.get("input_tokens", 0) or 0`. For requests with no `iterations`, `prompt_tokens` already excludes cache tokens (it was set from `_get_token_counts` which keys off `usage_object["input_tokens"]`), so subtracting cache tokens *again* will under-report. This is the most important issue in the PR — either gate the rewrite on `if iterations:` or verify the upstream `_get_token_counts` semantics include cache tokens (the test at `test_anthropic_compaction_usage_with_cache` only exercises the iterations branch, so this regression isn't caught).
- `types/utils.py:1681-1684` — the rename `self._cache_read_input_tokens` → `self._prompt_cache_hit_tokens` looks **unrelated** to compaction iterations and very likely breaks any downstream code reading `_cache_read_input_tokens`. This needs to either be split out into its own PR with justification, or reverted from this one. It's the kind of drive-by rename that causes silent regressions.
- `types/utils.py:1685-1686` — `if iterations is not None: self.iterations = iterations` is redundant with the field default (`None`) — `setattr` in the params loop below would handle it. Fine, but minor.
- `tests/test_anthropic_compaction_usage.py:1-2` — `import sys, os` is unused; `litellm` is also imported but unused. Remove for hygiene.
- The tests live at `tests/test_anthropic_compaction_usage.py` rather than `tests/test_litellm/llms/anthropic/chat/`. The latter is where this codebase puts Anthropic chat tests; relocate for discoverability and CI inclusion.
- Test coverage gap: no test asserts the *non-iterations* path is unchanged. Given the line 1774 rewrite, this is exactly the test that would catch the regression I'm worried about.

## Verdict: `request-changes`

The iterations-summing logic is correct and addresses Issue #27060. But the PR has two issues that must be fixed before merge: (1) `raw_input_tokens` rewrite at line 1774 changes behavior for *all* calls, not just iteration-bearing ones, with no test coverage on the non-iterations path; (2) the `_cache_read_input_tokens` → `_prompt_cache_hit_tokens` rename in `types/utils.py` is unrelated, unmotivated by the PR description, and is the kind of silent breakage that should never ship in a feature PR. Gate the rewrite, drop the rename (or split it), relocate the tests, add a non-iterations regression case.
