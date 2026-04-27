# BerriAI/litellm #26585 — fix(guardrails): re-emit chunks in tool_permission streaming hook when no tool_calls found

- **Repo**: BerriAI/litellm
- **PR**: #26585
- **Author**: someswar177
- **Head SHA**: 1facefea760e8298018ec1c0652dd65600105793
- **Size**: +45 / −0 across two files (one prod, one test).

## What it changes

Five-line bug fix in `litellm/proxy/guardrails/guardrail_hooks/tool_permission.py:683-691`
plus a 40-line regression test pinning the contract.

The pre-fix bug: `async_post_call_streaming_iterator_hook` is
an `async def` generator that, after consuming the upstream
stream into an `assembled_model_response`, branches on
"does the assembled response contain tool_calls?". When the
LLM replied with **plain text** (no tool calls), the
`if not tool_calls:` branch executed `return` — but `return`
inside an `async def` generator just terminates the
generator without yielding anything. The proxy then sent
clients `data: [DONE]` with zero content frames in between.
Every plain-text response from a guardrail-enabled LLM
endpoint dropped to an empty body.

The fix wraps the assembled response in a `MockResponseIterator`
and re-emits its chunks before `return`:

```python
mock_response = MockResponseIterator(
    model_response=assembled_model_response
)
async for chunk in mock_response:
    yield chunk
return
```

## Strengths

- **Correct minimum-surface fix.** The bug is a dropped
  payload with no error signal; the fix is to put the
  payload back. No new code paths, no config knobs, no
  guardrail-policy semantics changed — the no-tool-calls
  branch was previously a silent terminate and is now a
  re-emit-then-terminate.
- **`MockResponseIterator` is the right tool.** It already
  exists in litellm's chunk-builder utilities for the
  inverse direction (synthesize stream chunks from a
  collected `ModelResponse`). Reusing it here keeps the
  emitted chunk shape consistent with what a real upstream
  stream would have produced — the client won't see any
  difference between "guardrail passthrough" and "real
  upstream stream".
- **Strong regression test** at
  `tests/.../test_tool_permission.py:492-532`. The test
  patches `litellm.main.stream_chunk_builder` to return a
  fake assembled response, runs the hook over a synthetic
  stream of one plain-text chunk, and asserts
  `len(chunks) >= 1` with the load-bearing comment
  "Hook must yield at least one chunk for plain-text
  responses; got none — bare return bug". That comment
  alone is worth the diff — it pins the inverted-contract
  failure mode by name so any future "simplification" PR
  that yanks the re-emit fails fast with a docstring that
  explains why.

## Concerns / nits

- **Assertion is `>= 1`, not `== expected_chunk_count`.**
  The test would pass if the hook re-emitted *one* chunk
  even though the assembled response represents many
  upstream chunks. That's safe for the bare-return
  regression (`>= 1` vs `0` is the difference) but a
  follow-up assertion that the *content* of the emitted
  chunks matches the upstream payload (e.g. concatenated
  `chunk.choices[0].delta.content == "Hello, world!"`)
  would catch a future regression where someone replaces
  `MockResponseIterator` with a stub that yields one
  empty chunk.
- **The fix only covers the "no tool_calls" branch.** The
  diff hunk shows the `if not tool_calls:` guard at
  `:683`; the `else` branch (tool_calls present) is
  untouched, presumably because it goes through the
  permission-evaluation path that has its own emit logic.
  Worth confirming in the PR body that the
  permission-modify-and-emit path doesn't have the same
  bare-`return` shape lurking elsewhere — `rg "^\s*return\s*$" tool_permission.py`
  would surface any.
- **No log line on the re-emit path.** The pre-fix code
  had `verbose_proxy_logger.debug("Tool Permission Guardrail: No tool uses found")`
  on the same branch and the fix preserves it, but adding
  a one-line `verbose_proxy_logger.debug("Re-emitting %d chunks for plain-text passthrough", ...)`
  would help operators distinguish "passthrough fired" from
  "permission-modification fired" in trace logs.

## Verdict

**merge-as-is.** Right fix for a high-impact silent-drop
bug, with a regression test whose docstring pins the
failure mode by name. The content-match and elsewhere-bare-`return`
follow-ups are nice-to-haves, not blockers.

## What I learned

`async def` generators have two distinct termination
shapes that look identical in source — a bare `return`
ends the generator without yielding, and a `raise StopAsyncIteration`
does the same — neither produces a useful runtime signal.
When the function's contract is "consume an upstream
stream and emit chunks", any branch that doesn't yield
on its happy path is an empty-body bug waiting to be
discovered by a real client. Defensive review pattern:
in any `async def` generator that's contracted to relay
content, every reachable branch should either `yield`
at least once or `raise` an explicit error — never a
bare `return`. A linter rule keyed on this pattern
would have caught this PR's predecessor at write time.
