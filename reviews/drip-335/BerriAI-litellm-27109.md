# Review: BerriAI/litellm #27109

- **Title:** feat(mcp): split core mcp files into separate PR
- **Head SHA:** `932dd1043fa5ea0f84400788b869688627c2ff10`
- **Size:** +1442 / -471
- **Files touched (sample):**
  - `litellm/experimental_mcp_client/client.py`
  - `litellm/proxy/_experimental/mcp_server/elicitation_handler.py`
  - `litellm/proxy/_experimental/mcp_server/mcp_server_manager.py`
  - `litellm/proxy/_experimental/mcp_server/sampling_handler.py`
  - `litellm/proxy/_experimental/mcp_server/server.py`
  - `litellm/proxy/_types.py`

## Critique

Title says "split core mcp files into separate PR" but the diff is a large additive feature drop: sampling/elicitation/logging callback support on `MCPClient` plus new `sampling_handler.py`, `elicitation_handler.py` modules, plus refactoring of `client.py`. The "split" framing is misleading for reviewers — this is closer to a feature PR with a refactor included.

Specific lines:

- `client.py:5-15` — diff shows a large block of `-` lines that are pure whitespace deletions (removing blank lines after imports). Visually noisy; reviewers will struggle to tell which `-` lines are semantic. Recommend separating cosmetic blank-line removal into its own commit.
- `client.py:90-104` — new `sampling_callback`, `elicitation_callback`, `logging_callback` constructor params, all `Optional[Callable]`. Type hints are loose — `Callable` with no parameter signature means downstream callers get no type safety. Tighten to `Callable[[SamplingMessage, ...], Awaitable[SamplingResult]]` style or define `Protocol` classes; otherwise the whole MCP callback contract is implicit.
- `client.py:127` — change from `env=self.stdio_config.get("env", {})` to `env=self._get_safe_stdio_env(self.stdio_config.get("env"))`. This is a security-relevant change (env sanitization for stdio MCP servers). The `_get_safe_stdio_env` implementation is not visible in the first 150 lines of diff — reviewer must verify it doesn't strip required vars (PATH, HOME) and doesn't leak secrets through logging. Critical to confirm.
- New `elicitation_handler.py` and `sampling_handler.py` — adding handler modules with callback indirection requires test coverage. PR description doesn't reference test additions; verify the diff includes `tests/mcp_tests/` or similar.
- `litellm/proxy/_types.py` — touching shared `_types.py` in a feature PR is risky for downstream code that imports types. Confirm the changes are additive only (new types, no signature changes to existing ones).
- Aggregate +1442/-471 across MCP server + client: too large for a single review pass. Worth requesting the author actually do the split the title promises.

## Verdict

`request-changes`
