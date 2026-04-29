# BerriAI/litellm#26788 — fix(anthropic, mcp): sanitize tool names to match Anthropic's `^[a-zA-Z0-9_-]{1,128}$` pattern

- **PR**: https://github.com/BerriAI/litellm/pull/26788
- **Head SHA**: `8616d80ea99150eddb8f099ab325afb3a62e96f5`
- **Size**: +1028 / −15, 7 files
- **Files**: `litellm/llms/anthropic/chat/{transformation,handler}.py`, `litellm/proxy/_experimental/mcp_server/{openapi_to_mcp_generator,rest_endpoints}.py`, plus 3 test files (471 + 143 + 71 = 685 test lines).

## Context

When a user routes an OpenAPI-generated MCP tool through litellm to Anthropic, tool names like `actions/download-job-logs-for-workflow-run` get rejected upstream with `tools.28.custom.name: String should match pattern '^[a-zA-Z0-9_-]{1,128}$'`. The `/` and `.` characters Anthropic forbids are common in OpenAPI-derived `operationId`s. The PR adds two layers of defense:

1. **Provider-layer (Anthropic transformation)** — sanitize at request time, build a per-request reverse map, and translate tool-call names back to the caller's originals on the response side (both streaming and non-streaming).
2. **Generator-layer (OpenAPI → MCP)** — sanitize at registration time so MCP tools come out valid for *any* strict-name provider, not just Anthropic.

## Why this exists / problem solved

The key correctness insight, called out at `transformation.py:96-104` and elaborated in the PR body, is that **a bare `re.sub` is not safe** because sanitization is lossy: `foo/bar` and `foo_bar` both collapse to `foo_bar`. If you then reverse-mapped naively on the response side, a tool legitimately named `foo_bar` would get incorrectly retyped to `foo/bar`. The PR's fix is to build the forward map **per request** and only populate the reverse map for entries that were *actually rewritten* — names already valid pass through and are never reverse-mapped.

This is the right framing. Most OSS shims for Anthropic's name constraint either crash, error, or silently break round-trip for legitimately-named tools. This PR is the first I've seen articulate the round-trip safety property explicitly.

## Design analysis

### Forward/reverse map construction (`transformation.py:117-167`)

`_build_anthropic_tool_name_maps(original_names)` returns `(forward, reverse)`:

- `forward[original] = sanitized` only for rewritten names.
- `reverse[sanitized] = original` is `{v: k for k, v in forward.items()}`.
- Collision disambiguation uses numeric suffixes `_2`, `_3`, ... in insertion order.
- The collision check at `:160` uses a `used` set seeded with already-valid names, so a sanitization candidate that would collide with an already-valid name in the same request gets disambiguated. Good.

Two places worth scrutinizing:

1. **The suffix budget at `:163-164`** truncates to `_ANTHROPIC_TOOL_NAME_MAX_LEN - len(suffix)` then appends the suffix. For names already at the 128-char cap, this works. But for *original* names exceeding 128 chars where the sanitized candidate is also at the cap, two such originals will collide on the truncated head — the second gets `_2` and the suffix-aware truncation kicks in. Correct, but worth a unit test for "two originals >128 chars sharing the first 126 chars" — the diff has a unicode/length test but I didn't see this exact corner.

2. **Insertion-order dependence.** The PR docstring at `:140-143` correctly calls this out: "the second one seen gets the disambiguating suffix; callers should preserve the caller's tool order (we do)." Verify that *both* the streaming and non-streaming code paths iterate `tools` in the same order they were originally received — if any future refactor sorts tools alphabetically or by hash before passing to `_build_anthropic_tool_name_maps`, two clients with the same tool set but different orderings would see different sanitized names, which would break their multi-turn message rewriting at `_rewrite_tool_names_in_messages`.

### Forward-map application (request side)

The forward map is threaded through:

- `_map_tool_helper` at `transformation.py:566-571` — applies to `tool["function"]["name"]` when emitting `AnthropicMessagesTool`.
- `_map_tools` at `:797-803` — fans the forward map to each tool.
- `_map_tool_choice` at `:506-512` — applies to `tool_choice.function.name` so `tool_choice` matches the sanitized name on the wire.
- `_rewrite_tool_names_in_messages` (signature visible at `:809`, body cut off in my diff window) — rewrites historical assistant tool_calls in `messages` so multi-turn conversations stay consistent. **Crucial**: without this, a turn-2 assistant message referencing `actions/x` would conflict with a turn-1 tool definition that's been renamed to `actions_x`.

The `_apply_anthropic_tool_name_forward` helper at `:174-181` is a `if forward and name in forward: return forward[name] else: return name` no-op pass-through, which is the right shape — call sites unconditionally route through it without needing to check whether sanitization happened.

### Reverse-map threading (response side)

The reverse map is stashed in `litellm_params["_anthropic_tool_name_map"]` (constant `ANTHROPIC_TOOL_NAME_REVERSE_MAP_KEY` at `:115`) and threaded through:

- **Non-streaming**: `transform_response` → `transform_parsed_response` rewrites `tool_use` block names. (Code path described in PR body, not visible in my diff window.)
- **Streaming**: `make_call`/`make_sync_call` at `handler.py:86,146,244-249,471-476` accept `tool_name_reverse_map` and pass it to `ModelResponseIterator.__init__` at `:543`. The iterator stores it at `:551` and applies it inside `chunk_parser` at `:817-826` when emitting `tool_use` and `server_tool_use` content blocks.

The streaming code at `handler.py:817-826` is the most failure-sensitive part:

```python
_stream_tool_name = content_block_start["content_block"]["name"]
if (
    self.tool_name_reverse_map
    and _stream_tool_name in self.tool_name_reverse_map
):
    _stream_tool_name = self.tool_name_reverse_map[_stream_tool_name]
```

This is correctly conditional — empty map (the common case) is a single dict-truth check and the lookup is short-circuited, so the perf cost on requests without rewriting is one branch. Good. The replacement only happens for `tool_use` and `server_tool_use` block-start events, and the subsequent `content_block_delta` accumulation chunks don't carry the name (only the `id`/index), so we don't need to rewrite there.

One thing I'd verify: the stash-in-`litellm_params` plumbing assumes `litellm_params` is always a dict at the call sites at `handler.py:244-248` and `:471-475`. The defensive `if isinstance(litellm_params, dict) else None` handles the not-a-dict case by passing `None` (falling back to no rewriting). That's safe — worst case, the user sees the sanitized names in their response, which is the *current* behavior anyway, not a regression.

### Generator-side fix (`openapi_to_mcp_generator.py`)

`sanitize_openapi_tool_name` is a separate function (lossy `re.sub` + lowercase + truncate) applied at MCP-tool registration time. This is independent of the per-request map at the Anthropic transformation layer — generated MCP tools come out with valid names *upstream* of any provider transformation, so they're valid for OpenAI, Anthropic, Gemini, Cohere, and any future strict-name provider.

This raises a small architectural question: with the generator-side fix in place, will any Anthropic request actually need the per-request map? Answer: **yes**, because users can register MCP tools through other paths (raw API, third-party MCP servers, hand-written tool definitions) that don't go through `register_tools_from_openapi`. The two layers are complementary and both needed.

### Tests

The PR adds 685 lines of tests across three files:

- `test_anthropic_chat_transformation.py` (471 lines) — covers `_basic_sanitize_anthropic_tool_name` (special chars, length, unicode), `_build_anthropic_tool_name_maps` collision behavior (the `foo_bar` + `foo/bar` case is the load-bearing one), `_rewrite_tool_names_in_messages` (rewrites historical tool_calls, leaves unmapped names alone), and end-to-end `transform_request` + `transform_parsed_response` + streaming `chunk_parser`.
- `test_openapi_to_mcp_generator.py` (143 lines) — `sanitize_openapi_tool_name` plus `register_tools_from_openapi` for OpenAPI specs with invalid `operationId` and missing `operationId`.
- `test_rest_endpoints.py` (71 lines) — preview endpoint sanitizes consistently with registration.

PR body claims "238 passed" locally. That count should include pre-existing tests in those files; the new tests are likely ~30-40 cases across the three files.

## Risks

- **Multi-turn name churn.** If a user changes their tool definitions between turns (adds/removes a tool that participated in a collision), the forward map for the new request may assign different `_2` suffixes than the previous request. The historical `tool_calls` in `messages` carry the *old* sanitized names, but `_rewrite_tool_names_in_messages` rewrites using the *new* forward map. If the new map doesn't contain the old original (because the collision partner is gone), the old sanitized name passes through *without* getting rewritten — and Anthropic will see a mismatch between historical assistant tool_call names and current tool definitions. This is a subtle multi-turn correctness hole that the test suite should explicitly cover.
- **`tool_choice` for a removed tool.** If a request specifies `tool_choice={"type": "function", "function": {"name": "foo/bar"}}` but `foo/bar` isn't in `tools`, `_apply_anthropic_tool_name_forward` returns the name unchanged (no map entry). Anthropic will then 400 on the slash. Worth either pre-sanitizing `tool_choice` regardless of map presence, or documenting that `tool_choice` names must be in `tools` (which is already required by spec).
- **`_anthropic_tool_name_map` key collision.** The `litellm_params["_anthropic_tool_name_map"]` key is a private string. If any other litellm code path uses the same key (extremely unlikely but worth a `git grep`), there'd be silent data corruption. The leading underscore + provider prefix makes this a one-in-a-million collision; non-blocking.
- **No max-tools cap.** With 128 tools all sharing the prefix `foo/bar` (admittedly contrived), the suffix would grow to `_128`. The map construction is O(n²) in the worst case (linear scan of `used` set per name) but the inner loop is set-membership check which is O(1) amortized — actually O(n) total. Fine for any realistic tool count.
- **Static type interaction.** `_anthropic_tool_name_map` is stuffed into `litellm_params` which is typed loosely (`Dict[str, Any]` or similar). If anyone wraps `litellm_params` in a `TypedDict` later, this key won't be in the schema. Future-refactor concern; not blocking.

## Suggestions

1. **Add a multi-turn churn test** that covers: turn 1 has `[foo/bar, foo_bar]` (sanitized to `foo_bar`, `foo_bar_2`), turn 2 has only `[foo_bar]` (sanitized to `foo_bar`, no map entry). Assert that the historical assistant tool_call referencing `foo_bar_2` (from turn 1) is correctly *unmapped* in the turn-2 send (because it no longer exists) — or at least documents the expected behavior. This is the load-bearing edge case for multi-turn correctness.
2. **Sanitize `tool_choice` unconditionally**, not just when there's a forward-map hit. At `transformation.py:506-512`, even if `_tool_name` isn't in the forward map, run it through `_basic_sanitize_anthropic_tool_name` so a stray slash in `tool_choice` doesn't 400. Belt-and-suspenders.
3. **Document the suffix policy** in user-visible release notes. Users with metric/log dashboards keyed on tool names will see `foo_bar_2` appearing where they expected `foo/bar`. The "you'll see sanitized names in any external observability that reads from Anthropic-side payloads" footnote is worth a CHANGELOG line.
4. **Consider extracting `_build_anthropic_tool_name_maps` to a provider-agnostic helper** in `litellm/utils/tool_name_sanitization.py`, parameterized on the regex pattern and max length. Other providers (e.g., Vertex's tool-name limits, future strict providers) can reuse the same per-request collision-safe map construction without re-implementing the logic. Anthropic gets a thin wrapper that pins `[a-zA-Z0-9_-]` + 128.
5. **Split the PR for review reviewability.** 1028 lines in one PR is a lot for a transformation change; the generator-side fix at `openapi_to_mcp_generator.py` could land first as a standalone (it has its own 143-line test file), then the Anthropic transformation layer as a separate PR with a smaller blast radius. Maintainer's call — current scope is reviewable but on the high side.

## Verdict

**merge-after-nits** — the round-trip-safety design is correct (the `forward`/`reverse` asymmetry is the key insight), the threading through both streaming and non-streaming paths is consistent, and the test coverage on the load-bearing pieces (`_build_anthropic_tool_name_maps` collision, end-to-end streaming chunk_parser reverse) is solid. The asks are: (1) close the multi-turn churn correctness hole with an explicit test, (2) sanitize `tool_choice` unconditionally, (3) CHANGELOG note about user-visible name churn in observability. None are blocking on the basic functionality but #1 in particular is a real correctness gap.

## What I learned

The "lossy sanitization + naive reverse map" anti-pattern is everywhere in LLM-shim code (it's how most OpenAI-compat shims handle restrictions), and this PR articulates the right correction crisply: **only reverse-map names you actually rewrote**. The trick is keeping the forward and reverse maps in lockstep — if you ever populate `reverse[sanitized] = original` without checking that `sanitized != original`, you've reintroduced the bug. The Pythonic `reverse = {v: k for k, v in forward.items()}` at `:166` is correct precisely because `forward` only contains rewritten entries by construction; the round-trip safety is enforced at population time, not lookup time. That's the kind of property worth pinning with a test that asserts `forward.keys()` and `reverse.values()` are disjoint from any "untouched" set, so a future refactor can't accidentally weaken the invariant.
