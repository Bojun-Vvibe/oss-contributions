# BerriAI/litellm #26868 — fix(embedding): respect drop_params for unsupported dimensions parameter

- **Author:** xr843
- **SHA:** `abf3898`
- **State:** OPEN
- **Size:** +75 / -11 across 2 files
  (`litellm/utils.py`,
  `tests/local_testing/test_get_optional_params_embeddings.py`)
- **Verdict:** `merge-as-is`

## Summary

Closes #26787. The OpenAI-provider branch in `get_optional_params_embeddings`
at `litellm/utils.py:3309-3327` raised `UnsupportedParamsError` whenever
`dimensions` was passed to a non-`text-embedding-3` model — even though the
error message itself instructed users to set `litellm.drop_params=True`. The
flag (per-call and global) had no effect, so the documented escape hatch was
a dead end. Fix replaces the unconditional `raise` with an `if
litellm.drop_params is True or (drop_params is not None and drop_params is
True): non_default_params.pop("dimensions", None)` arm matching the
convention used by `_check_valid_arg` earlier in the same function. Locked by
two new tests + a hardened existing test covering all three flag
combinations: per-call `drop_params=True` at `:107-127`, global
`litellm.drop_params=True` at `:130-152`, and the existing raise path
hardened at `:97-114` to explicitly reset `litellm.drop_params = False`
before asserting the raise (because the file's other tests flip the global,
making test order load-bearing without the reset).

## Reasoning

This is a genuinely tight fix on a high-traffic surface with three
correctness arguments worth calling out:

1. **The fix matches the convention already established in the same
   function.** `_check_valid_arg` earlier in `litellm/utils.py` already
   honors the same `litellm.drop_params is True or drop_params is True`
   predicate; the OpenAI-embedding branch was the outlier. Aligning it
   removes a "why does drop_params work here but not there" footgun for
   the next reader and matches the pattern users already document
   internally.

2. **The structural change at `:3324` (moving `optional_params =
   non_default_params` *outside* the `if/else`) is correct.** Before, the
   assignment lived in the `else` arm of the `raise`, which was a
   dead-code-path-by-construction (because the `if` raised). After, the
   `if`/`elif`/`else` shape collapses to "drop the unsupported param OR
   raise", and the assignment after the conditional captures the
   (possibly-mutated) `non_default_params`. This is the right cleanup.

3. **The existing test was correctly hardened.** The previous
   `test_openai_non_text_embedding_3_without_allowed_openai_params_raises`
   was order-dependent — earlier tests in the file set
   `litellm.drop_params = True` and never restored it, so this test
   passed only if pytest ran it before the global was flipped. The new
   `prev_drop_params = litellm.drop_params; litellm.drop_params = False;
   try: ... finally: litellm.drop_params = prev_drop_params` envelope
   makes the test order-independent. Both new tests use the same
   try/finally restore pattern, so the file is now hermetic regardless of
   pytest run order or `pytest -k` selection.

4. **Test naming follows the established `test_<area>_<scenario>` pattern
   from the rest of the file and the docstrings explicitly cite issue
   #26787.** That gives the next person staring at a CI failure exactly
   the breadcrumbs they need to understand the regression in one click.

The `non_default_params.pop("dimensions", None)` shape (with the explicit
`None` default) is correctly defensive — although today the code only
reaches the pop when `"dimensions" in non_default_params.keys()` was already
true, the explicit default protects against a future refactor that hoists
the pop outside the keys-check.

Net: minimal change to a small surface, fixes a concrete user-reported bug
where the documented escape hatch was lying, ships with three regression
tests covering all three flag combinations and a fourth hardening of the
raise path. Mergeable as-is.
