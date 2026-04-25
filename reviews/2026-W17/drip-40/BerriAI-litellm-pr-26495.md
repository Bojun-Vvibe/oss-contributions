# BerriAI/litellm #26495 — fix(health_check): drop max_tokens from non-chat handler params (closes #26406)

- **Repo**: BerriAI/litellm
- **PR**: [#26495](https://github.com/BerriAI/litellm/pull/26495)
- **Head SHA**: `bd3afc6493da493fa853a6d5f4995f03bf20262e`
- **Author**: he-yufeng
- **State**: OPEN (+9010 / -863, but ~80 lines of net fix; remainder
  is a dev-branch merge sweep)
- **Verdict**: `merge-after-nits`

## Context

`/health` for OpenAI image-generation deployments
(`dall-e-2`, `dall-e-3`, `gpt-image-1`) was hard-failing with
`400 Unknown parameter: 'max_tokens'`, leaving the proxy
permanently reporting them as unhealthy (#26406). The bug was
masked on providers whose transformers silently drop unknown
fields (Vertex), which is why it took a while to surface.

Root cause: `_update_litellm_params_for_health_check`
(`litellm/proxy/health_check.py`) injects `messages` **and**
`max_tokens` into every deployment's `litellm_params` before
dispatch. The shared sanitiser
`_filter_model_params` only stripped `messages`, so
non-chat handlers (image / video generation, embedding,
transcription, rerank, ocr, batch, audio_speech) all inherited
the spurious `max_tokens`.

## Design

Two coordinated edits.

1. **`litellm/litellm_core_utils/health_check_utils.py:5-22`**
   widens `_filter_model_params` from a hardcoded `messages`
   strip into a strip-set:

   ```python
   def _filter_model_params(
       model_params: dict, additional_keys_to_remove: Optional[Iterable[str]] = None
   ) -> dict:
       keys_to_remove = {"messages"}
       if additional_keys_to_remove:
           keys_to_remove.update(additional_keys_to_remove)
       return {k: v for k, v in model_params.items() if k not in keys_to_remove}
   ```

   `messages` stays default — every existing caller keeps its
   contract.

2. **`litellm/litellm_core_utils/health_check_helpers.py:152-220`**
   defines a `_non_chat_filter()` closure inside
   `get_mode_handlers` that pre-binds
   `{"max_tokens"}` and threads it through every non-chat
   handler:

   ```python
   non_chat_extra_keys = {"max_tokens"}

   def _non_chat_filter() -> dict:
       return _filter_model_params(
           model_params=model_params,
           additional_keys_to_remove=non_chat_extra_keys,
       )
   ```

   Then `embedding`, `audio_speech`, `audio_transcription`,
   `image_generation`, `video_generation`, `rerank`, `ocr`,
   and `batch` (via the `filtered_model_params=` kwarg) all
   call `_non_chat_filter()` instead of the bare
   `_filter_model_params`. The `chat` and `responses` handlers
   keep the original `_filter_model_params(...)` — chat
   obviously needs `max_tokens`, and the Responses API still
   accepts it. That's the right line to draw.

3. The `audio_speech` branch's old "compute filter twice to
   check for `voice`" pattern collapses correctly:

   ```python
   "audio_speech": lambda: litellm.aspeech(
       **{
           **_non_chat_filter(),
           **({"voice": "alloy"} if "voice" not in _non_chat_filter() else {}),
       },
       input=prompt or "test",
   ),
   ```

   It's still calling `_non_chat_filter()` twice per dispatch;
   could be a one-line `_filtered = _non_chat_filter()`
   binding above the lambda dict. Pre-existing inefficiency,
   not introduced here, but worth fixing while in the
   neighborhood.

## Risks

- **The `responses` handler at `health_check_helpers.py:215`
  still uses the bare `_filter_model_params(model_params=model_params)`.**
  That's correct for the OpenAI Responses API (which accepts
  `max_tokens`), but if any third-party "responses-shaped"
  provider routes through this same path and rejects
  `max_tokens` for non-text outputs, the same bug recurs there.
  Worth a one-line comment at line 215 noting why `responses`
  is intentionally not on `_non_chat_filter()`.
- **The PR is +9010 / -863.** Most of that is a dev-branch
  resync (CI configs, vertex docs, `prompt_templates`,
  `logging_worker.py`, anthropic adapter, dozens of unrelated
  files). The actual fix is ~80 lines across two files. This
  is a **review hazard**: a future bisect against this PR will
  point at the wrong file half the time. Strongly recommend
  rebasing onto a clean `main` and force-pushing, or splitting
  the fix off into a new PR — otherwise the merge commit will
  be cited as the cause of unrelated regressions.
- **No new test pinning the failure mode.** The fix is small
  enough that a unit test against
  `_filter_model_params(model_params={"messages": [...], "max_tokens": 100, "voice": "alloy"}, additional_keys_to_remove={"max_tokens"})`
  asserting the result equals `{"voice": "alloy"}` would lock
  this in for ~5 lines. The bug was a silent contract
  violation; without a test, the next refactor of the helpers
  module reintroduces it.

## Suggestions

- Hard-rebase to drop the 8800 lines of unrelated drift before
  merge. The actual fix is exactly the two-file diff in
  `health_check_utils.py` + `health_check_helpers.py`.
- Add a unit test in
  `tests/litellm_utils_tests/test_health_check_utils.py`
  exercising both the default (`messages` only) and extended
  (`messages + max_tokens`) strip behaviour.
- Bind `_filtered = _non_chat_filter()` once for the
  `audio_speech` handler instead of computing it twice in the
  dict expression.
- Add a comment on the `responses` handler explaining why it
  keeps `max_tokens` (so the next reader doesn't "fix" it
  symmetrically and break the Responses path).

## Verdict reasoning

`merge-after-nits`. The behaviour change is small, surgical,
and correct. The blocker is the dev-branch merge noise — that
needs to come out before merge so the PR represents the actual
fix. Test + audio_speech micro-cleanup are nice-to-haves.

## What I learned

The "additional_keys_to_remove" parameter is a clean way to
extend a shared sanitiser without forking it — every existing
caller keeps its contract because the default behaviour is
preserved, and the new caller opts into the wider strip
explicitly. The wrong instinct here would have been to add
`_filter_model_params_for_non_chat()` as a sibling function;
that ends up with two near-identical functions that drift
apart over time. The closure-around-a-strip-set pattern keeps
the call sites readable (`_non_chat_filter()` reads as
intent, not mechanism) and centralises the strip set in one
place. Worth lifting `non_chat_extra_keys` to a module-level
constant so a `BATCH_HEADER_FIELDS` audit can grep for it.
