# BerriAI/litellm#26636 — fix(anthropic): preserve all tool_calls when an OpenAI delta contains multiple

- **PR**: https://github.com/BerriAI/litellm/pull/26636
- **Author**: freddyhaddad (Frederic Haddad)
- **Head SHA**: `2e3da159be8adcbcc3ed3af38271eee63768f1e9`
- **Verdict**: `merge-after-nits`

## Context

The Anthropic experimental pass-through adapter at `litellm/llms/anthropic/experimental_pass_through/adapters/streaming_iterator.py` translates an upstream OpenAI-format streaming completion into Anthropic's `/v1/messages` SSE shape. The downstream converter `_translate_streaming_openai_chunk_to_anthropic_content_block` in `AnthropicStreamWrapper` indexes `tool_calls[0]` — i.e. it assumes one tool_call per OpenAI delta. Most OpenAI-compatible providers honor that one-call-per-delta convention, but `mlx_lm.server` (and presumably any other provider that emits parallel function calls in a single delta) violates it: when the model decides to call N functions in parallel, the provider emits one `delta.tool_calls = [tc1, tc2, ..., tcN]` chunk, and the adapter silently drops `tc2..tcN`. The user-visible symptom is "Anthropic-format consumer sees only one of N parallel tool calls and the others vanish without an error."

## What it changes

Introduces a new `_MultiToolCallSplitter` class at `:30-86` that wraps the upstream completion stream and emits at most one tool_call per delivered chunk by splitting any incoming chunk whose `delta.tool_calls` has length > 1 into N chunks (one tool_call each, via `copy.deepcopy`). The splitter implements both sync (`__iter__` / `__next__`) and async (`__aiter__` / `__anext__`) protocols and lazily binds whichever one is invoked first — necessary because `AnthropicStreamWrapper` exposes both `__next__` and `__anext__` and consumers pick the matching protocol. A static helper `_split_chunk_by_tool_calls(chunk)` at `:113-140` does the actual partitioning: returns `[chunk]` for `None`, `"None"`, missing-choices, or 0/1 tool_calls; otherwise deep-copies the chunk per tool_call and replaces `sub.choices[0].delta.tool_calls` with a deep-copied single-element list. The splitter is stored on a new attribute `self._completion_stream_splitter` at `:111` rather than overwriting the parent's `self.completion_stream` (so consumers that read the original stream still see it), and the iteration site at `:230` is changed from `for chunk in self.completion_stream` to `for chunk in self._completion_stream_splitter`.

## Design analysis

Three things are right and worth naming. First, the **dual-protocol lazy binding** at `:60-86` solves a real problem — the upstream `completion_stream` may expose both `__iter__` and `__aiter__` (because it's a wrapper class, not a generator), and binding to one at construction time would break whichever protocol is unused. Lazy-bind-on-first-call is the correct fix; the comment at `:53-56` names the contract well. Second, the **`copy.deepcopy` of both the chunk *and* each `one_tc`** at `:133-138` is load-bearing — without the inner deep-copy of `one_tc`, mutations downstream on `sub.choices[0].delta.tool_calls[0]` (e.g. argument-delta accumulation in subsequent chunks of the same tool-call sequence) would alias across split chunks and produce a confusing "the second tool call's arguments got appended to the first" bug that would only manifest for long argument streams. The comment at `:134-138` correctly names this. Third, the **separate-attribute storage** at `:111` (`self._completion_stream_splitter`) rather than overwriting `self.completion_stream` is the right shape for a subclass attribute — both static analyzers and any future consumer that walks `self.completion_stream` directly get the unmodified original.

## Concerns / nits

1. **`chunk == "None"` literal-string check at `:119`** is unusual — typically this means the upstream stream is yielding the literal string `"None"` somewhere as a sentinel, which suggests an upstream serialization bug that should be fixed at the source rather than papered over here. Worth a comment naming *why* this case exists (which upstream emits `"None"` as a string?) or, better, a TODO to find and fix the source.

2. **`_sync_iter_obj` and `_async_iter_obj` typed as `Any` at `:57-58`** with the comment that this is to avoid mypy narrowing. That's true but the type annotation `Optional[Any]` would be just as `Any`-equivalent for mypy and would document the "starts None, becomes the iterator after first call" lifecycle. The current `Any` typing makes a reader briefly believe these can never be None.

3. **No `__del__` / cleanup contract** — the splitter holds a reference to `self._stream` and to `self._sync_iter_obj` / `self._async_iter_obj`; if the upstream stream needs explicit close (some HTTP streams do), this wrapper doesn't propagate it. Worth a `close()` method that delegates to `self._stream.close()` if it exists, or at minimum a comment at the class docstring naming the assumption ("upstream stream lifetime is owned elsewhere").

4. **`copy.deepcopy(chunk)` at `:133` is potentially expensive** for chunks with large embedded payloads (e.g. function arguments that are themselves large JSON blobs). For the multi-tool-call case this is unavoidable (you have to deep-copy per tool_call), but for the common single-tool-call case the early-return at `:129-130` correctly bypasses it. The cost is bounded by N (number of parallel tool_calls in one delta), which is realistically ≤10. Worth a perf note in the docstring.

5. **No regression test in the diff visible at lines 1-150** — the `_split_chunk_by_tool_calls` static method is the unit-testable surface and it should have a test that pins the three branches: 1-call passthrough, N-call split with deep-copy independence (mutate one and assert the others don't move), and the `None` / `"None"` / no-`choices` no-op cases. Worth adding before merge.

## Risks

Medium. The change is correctness-positive for the parallel-tool-call case (which was previously silently broken), and the `len(tcs) <= 1` early-return at `:129` makes the splitter a no-op for the existing single-tool-call provider universe — so no regression risk for current users. The dual-protocol lazy binding is novel for this file and the failure mode if mis-implemented would be "consumer hangs forever" or "consumer raises `TypeError: 'X' object is not iterable`" — both would be caught by any existing test that exercises either protocol, but a targeted regression test for the splitter itself would be cheap insurance.

## Suggestions

Land after: (a) adding a unit test for `_split_chunk_by_tool_calls` covering passthrough, split, and the three no-op shapes; (b) replacing the `chunk == "None"` literal check with a comment naming the upstream source (or filing a TODO); (c) tightening the `_sync_iter_obj` / `_async_iter_obj` type annotation to `Optional[Any]` and dropping the comment. The deep-copy correctness is already well-handled and the dual-protocol design is sound.

## What I learned

The "dual-protocol lazy-binding wrapper" pattern (decide sync-vs-async on the first protocol invocation, not at construction) is the right shape for any iterator wrapper that sits between a class exposing both protocols and a consumer that picks one. Easier than splitting into `_SyncMultiToolCallSplitter` + `_AsyncMultiToolCallSplitter` factories, and avoids the "wrapper around a wrapper around a wrapper" depth that those factories tend to produce.
