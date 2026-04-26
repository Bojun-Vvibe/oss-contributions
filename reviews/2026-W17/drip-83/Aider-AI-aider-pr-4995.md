# Aider-AI/aider PR #4995 — fix: catch RuntimeError in command execution

- Repo: Aider-AI/aider
- PR: #4995
- Head SHA: `38e806fb`
- Author: @jjjojoj
- Diff: +5/-1 in `aider/coders/base_coder.py`
- Closes: #4988

## What changed

In `preproc_user_input` (`aider/coders/base_coder.py:914-921`), wraps `self.commands.run(inp)` in a `try / except RuntimeError` that calls `self.io.tool_error(f"Error running command: {err}")` and returns `None` instead of crashing aider's main loop. Targets the issue-4988 stack: HuggingFace embedding-model download surfacing `RuntimeError: Cannot send a request, as the client has been closed` through whatever command happened to trigger embedding load.

## Specific observations

- The catch is correctly scoped to `RuntimeError` only, not bare `Exception`. That's the right call for this fix because aider already has higher-level exception handling and broad-catching here would mask programming bugs (KeyError in command parsing, etc.). The PR description correctly identifies HuggingFace's "client has been closed" as the concrete failure mode, and that's a `RuntimeError`.
- Fix is at the wrong layer though, arguably. The actual bug is that an HF-embedding HTTP client is being held across operations and getting closed underneath callers. Catching `RuntimeError` in `preproc_user_input` makes the specific symptom non-fatal but leaves the closed-client state alive — the *next* command that hits the same client will hit the same `RuntimeError`. A more correct fix would be in `aider/repomap.py` (or wherever the embedding model is held) to detect the closed-client state and lazily re-init. Worth raising in review as "this is a band-aid; the underlying client lifecycle bug should also be filed."
- Sibling PR #4994 from the same author wraps `cmd_help` with the *same* `RuntimeError` catch pattern. Together they suggest the contributor is patching individual call sites against the same root cause. Maintainers should probably ask for one more iteration that catches the issue at the embedding-client boundary instead.
- No test added. A `test_preproc_user_input_runtime_error_does_not_crash` that monkey-patches `commands.run` to `raise RuntimeError` and asserts `tool_error` is called would lock the contract in cheaply.

## Verdict

`merge-after-nits`

## Rationale

The change is safe and small and prevents user-visible crashes for a real failure mode. It's a band-aid rather than a root-cause fix — happy to merge it as a stop-gap as long as a follow-up issue is opened to address the embedding-client lifecycle properly.
