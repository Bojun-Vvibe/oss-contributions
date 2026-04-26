# Aider-AI/aider PR #4994 — fix: handle errors in /help command gracefully

- Repo: Aider-AI/aider
- PR: #4994
- Head SHA: `7d94be54`
- Author: @jjjojoj
- Diff: targeted `aider/commands.py:1132-1168` (cmd_help)
- Closes: #4992

## What changed

Wraps three call sites in `cmd_help`:

1. `Help()` constructor (line 1135) — catches `(RuntimeError, OSError, Exception)` and emits `tool_error("Unable to initialize interactive help: {err}")`.
2. `Coder.create(...)` (lines 1141-1149) — catches `Exception` and emits `tool_error("Unable to initialize help coder: {err}")`.
3. `coder.run(user_msg, preproc=False)` (lines 1161-1165) — catches `Exception` and emits `tool_error("Error running help: {err}")`.

Each catch returns early instead of bubbling. Targets the `litellm.BadRequestError` (no LLM provider configured) and `RuntimeError` (HuggingFace embedding download failure) crash paths from #4992.

## Specific observations

- The first `except (RuntimeError, OSError, Exception)` (line 1135) is **redundant in a confusing way** — `Exception` is the parent class of both `RuntimeError` and `OSError`, so the tuple effectively means "catch Exception." Either narrow it to `(RuntimeError, OSError)` to actually limit scope, or simplify to just `Exception`. As written it suggests the author wasn't sure of the type hierarchy. Recommend narrowing to the two specific types that are documented to occur (`RuntimeError` from embedding load, `OSError` from disk/network), since broad `Exception` catch on `Help()` will swallow real programming bugs.
- The other two catches (`Coder.create`, `coder.run`) use bare `except Exception`, which is too broad. `Coder.create` can fail with `UnknownEditFormat` (already handled at the caller in `main.py:1010`), `litellm.BadRequestError`, etc. — narrowing to those specific types preserves diagnosability. Bare `Exception` will swallow `KeyboardInterrupt` only because `KeyboardInterrupt` derives from `BaseException`, not `Exception` — that's correct here, but assertion errors in tests will be hidden, which makes regressions harder to catch.
- Same band-aid pattern as #4995 from the same author: this fixes the user-visible crash but leaves the underlying "no LLM provider" / "embedding download fails" misconfigurations invisible-after-message. Worth pairing with a one-line docs note in the help text that says "if /help fails, run `aider --models` to confirm provider auth" so users have a recovery path.
- No tests. A `test_cmd_help_handles_help_init_failure` that monkey-patches `Help` to `raise RuntimeError` would be the minimum coverage to keep this from regressing.

## Verdict

`request-changes`

## Rationale

The intent is right — `/help` shouldn't take down aider's main loop — but the exception types are wrong in a way that obscures real bugs. Specifically: `(RuntimeError, OSError, Exception)` should be `(RuntimeError, OSError)`, and the bare `Exception` catches around `Coder.create` / `coder.run` should be narrowed to the documented failure types (`litellm.BadRequestError`, `RuntimeError`, `OSError`). Once those are tightened and one regression test is added, this is a clean merge.
