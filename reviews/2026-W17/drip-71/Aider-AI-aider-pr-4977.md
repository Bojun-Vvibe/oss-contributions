# Aider-AI/aider PR #4977 — abort cleanly in batch mode (--exit) when Ollama is unreachable

- **PR**: https://github.com/Aider-AI/aider/pull/4977
- **Author**: @Frank-Schruefer
- **Head SHA**: `84df006a2ea56f93529de98dd9364a887ec8a4ec`

## Summary

Three coordinated changes in `aider/main.py` and `aider/models.py`:

1. **`Model.__init__`** (`models.py:332`): adds `self.ollama_error = None` slot.
2. **`Model.get_model_info`** (`models.py:362-369`): wraps the call to
   `model_info_manager.get_model_info(model)` so that an exception
   whose `str()` starts with `"OllamaError:"` is captured into
   `self.ollama_error` and an empty info dict is returned (instead of
   re-raising). Other exception types still propagate.
3. **`ModelInfoManager.get_model_info`** (`models.py:245-247`):
   inside its existing broad `except Exception as ex:` block, an
   early `if str(ex).startswith("OllamaError:"): raise` ensures that
   Ollama errors are *not* swallowed by the existing
   `model_prices_and_context_window.json` fallback path — they
   propagate up so the outer wrapper above can capture them.
4. **`main.py:889-895`**: after model construction, if
   `main_model.ollama_error and args.exit`, emit a clear
   `io.tool_error("Cannot connect to Ollama: ... Aborting because
   --exit (batch mode) was specified.")` and `return 1`.

A test for `sanitize_for_utf8` is also visible in `tests/basic/test_models.py`
but it belongs to a different PR (#4987); this PR's changes are the
three above.

## Verdict: `merge-after-nits`

Right shape for a real CI/automation pain point: a batch-mode aider
invocation against an unreachable Ollama would previously print a
warning and *continue*, eventually trying to send a real LLM request
and failing later in a less actionable way. Now it exits 1 immediately
with a clear message.

## Specific references

- `models.py:245-247` — early-raise inside `get_model_info`. Order
  matters: it's *before* the `model_prices_and_context_window.json`
  fallback, so Ollama errors don't get suppressed by the JSON-fallback
  branch. Correct positioning.
- `models.py:362-369` — outer wrapper that converts the propagated
  exception into a captured `self.ollama_error` slot. Returning `{}`
  for `model_info` means downstream callers see "no info" rather
  than a hard failure, which preserves interactive-mode UX
  (interactive aider can still try to send a request and let the
  user see the connection-refused error in context).
- `main.py:890-895` — the `args.exit` gate. This is what makes the
  fix scope-correct: only batch mode aborts; interactive mode
  retains the older "warn and try anyway" behavior.

## Concerns

1. **String-prefix exception classification.** `str(ex).startswith("OllamaError:")`
   is fragile. A future litellm version that renames the exception
   to `OllamaConnectionError` or changes the prefix to
   `Ollama Error:` will silently regress. Prefer `isinstance(ex,
   litellm.exceptions.OllamaError)` (or whatever the canonical class
   is) with a graceful fallback to the prefix check for older
   litellm versions. At minimum, define the prefix once as a
   module-level constant `OLLAMA_ERROR_PREFIX = "OllamaError:"` and
   document where it comes from.

2. **`self.ollama_error = ex`** stores the exception object itself
   (line 367). The `main.py` formatter then does `f"... : {main_model.ollama_error}"`
   which will call `str(ex)` again. If the exception's `__str__`
   already includes `"OllamaError: "`, the user sees
   `"Cannot connect to Ollama: OllamaError: connection refused"` —
   slightly redundant. Strip the prefix when storing, or format
   without it.

3. **`get_model_info` returns `{}` on Ollama failure.** Downstream
   callers that index into the returned dict (e.g.
   `info["max_input_tokens"]`) will now get `KeyError` instead of
   the previous (crashing) behavior. Verify all call sites use
   `.get(...)` rather than `[...]` access. If not, this PR
   substitutes one crash for another.

## Nits

- The new `ollama_error` slot is set in `__init__` but only updated
  in `get_model_info`; if `get_model_info` is called multiple times
  the slot is overwritten each time. Acceptable but document.
- The `--exit` flag rationale could go into a one-line comment on
  the new `main.py` branch so the next reviewer doesn't have to
  guess why interactive mode is exempt.
