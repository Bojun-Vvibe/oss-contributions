# Review: BerriAI/litellm #27114 — feat(utils): sanitize OpenAI tool names

- **PR**: https://github.com/BerriAI/litellm/pull/27114
- **Author**: Sameerlite
- **Base**: `litellm_internal_staging`
- **Head SHA**: `833b62ca71d7c407abdda6a6abebd5aee7e98803`
- **Size**: +377 / −11 across 6 files

## Scope

OpenAI's Chat Completions schema requires `tools[].function.name` to match `^[a-zA-Z0-9_-]+$` (≤64 chars). When clients pass tool names with `.`, `:`, spaces, etc., upstream rejects the call. This PR rewrites tool names before the upstream call (only for providers that enforce the OpenAI rule) and restores the client-supplied names in the response so callers don't see the rewrites. Mirrors the existing Bedrock sanitization model.

## Substantive review

### `litellm/litellm_core_utils/openai_tool_name_mapping.py` (+76 new file)

- Storage is a `contextvars.ContextVar[Dict[str, str]]` named `litellm_openai_tool_name_mapping`. **Correct choice**: ContextVar is per-asyncio-task and per-thread, so concurrent completions cannot leak each other's mappings. The module-level docstring explicitly calls this out.
- `_OPENAI_TOOL_NAME_VALIDATION_PROVIDERS` (line ~22) frozenset enumerates: `openai`, `azure`, `azure_ai`, `custom_openai`, `text-completion-openai`, `groq`, `deepinfra`, `together_ai`, `fireworks_ai`, `nvidia_nim`, the GitHub assistant provider, `perplexity`, `xai`. **Concern**: this is an allow-list; any new OpenAI-compatible provider added later that also enforces the validation will silently fail until added here. A safer default for new OpenAI-compatible providers might be opt-out rather than opt-in.
- `begin_openai_tool_name_mapping_scope()` (line ~50) calls `_CTX.set({})` — fresh dict per completion. Good; no cross-request bleed.
- `_store(sanitized, original)` (line ~57) early-returns when `sanitized == original`, keeping the dict small. Good.
- `restore_openai_tool_name_for_user` is a thin wrapper — no concerns.

### `litellm/main.py` (+37 / −1)

- Lines ~1163–1196: wraps `validate_and_fix_openai_tools` with a `get_llm_provider` lookup to decide whether to sanitize. The `try/except: _sanitize_openai_fn_tool_names = False` swallow on line ~1175 is **too broad** — silently falling through to "don't sanitize" for any exception inside `get_llm_provider` will produce upstream 400s with confusing error text. At minimum, log the exception at DEBUG.
- The `tool_choice` rewriting block (line ~1183) imports `_sanitize_openai_function_tool_name` from `litellm.utils` lazily inside the function. Acceptable given the existing import-cycle landscape, but a top-of-file import after the fact would read better.
- The `tool_choice` rewrite calls `_san_name(_fn["name"], -1)` — the `-1` index parameter is opaque without seeing the helper signature; please confirm `-1` means "no collision suffix needed" or similar, and add an inline comment.

### `litellm/litellm_core_utils/llm_response_utils/convert_dict_to_response.py` (+16)

- New `_map_tool_call_dict_openai_names_to_user(tc)` (line ~74) mutates a copy of the tool-call dict, replacing `function.name` with the restored client name when the mapping has it. Implementation is `**tc, "function": {**fn, "name": restored}` — clean immutable update.
- The hook into the conversion loop (line ~563) runs only when `_tc` is a dict. Tool calls that arrive as Pydantic models won't be remapped here — please verify the streaming path (handled separately in `streaming_chunk_builder_utils.py`) covers the model-form case.

### `litellm/litellm_core_utils/streaming_chunk_builder_utils.py` (+6 / −1)

- Line ~299: applies `restore_openai_tool_name_for_user` when assembling the streamed combined tool-call. Symmetric with the non-streaming path. ✓

### `litellm/utils.py` (+81 / −6) and `tests/test_litellm/test_utils.py` (+161 / −3)

The diff window I have shows the test file adds 161 lines. Without the full body visible, I'm relying on the description ("collision suffixes when needed", "max 64 chars"). Please make sure the test suite covers:
- Names containing `.`, `:`, space, `/`, unicode characters.
- Names exceeding 64 chars (truncation behaviour).
- Two distinct client names that sanitize to the same string (collision suffix).
- Round-trip through both streaming and non-streaming response paths.
- ContextVar isolation between two concurrent `asyncio.gather` completions.

## Verdict

**merge-after-nits** — this is a high-quality, well-architected fix to a real interop problem; the ContextVar-scoped mapping is exactly right. Two concrete blockers before merge: (1) tighten the bare `except Exception` in `main.py` to log at DEBUG, and (2) confirm the test suite includes the concurrent-isolation case for the ContextVar mapping. The provider allow-list is a structural concern worth filing as a follow-up but is reasonable for the initial landing.
