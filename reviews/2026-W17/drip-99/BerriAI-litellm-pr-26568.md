# BerriAI/litellm#26568 — feat(arize): emit prompt-caching tokens as OpenInference span attributes

- **Head**: `d4efaf8cf9b52d5d5c9ec1a29858df6d30513647`
- **Size**: +209/-0 across 2 files
- **Verdict**: `merge-as-is`

## Context

Restores `cached_tokens` emission to Arize / OpenInference spans (which
PR #24112 added but a downstream refactor in #26506 inadvertently dropped
during a switch to raw OpenAI Pydantic usage support). Also adds
`cache_creation_tokens` emission, which has *never* been wired up
despite the `LLM_TOKEN_COUNT_PROMPT_DETAILS_CACHE_WRITE` constant being
defined in OpenInference. Net effect: observability backends like
Langfuse can finally display the cache-read / cache-write breakdown for
Anthropic, OpenAI, and DeepSeek prompt caching, and apply the correct
cost calculation (cache reads at 0.1×, cache writes at 1.25×).

## Design analysis

### Source of truth: `prompt_tokens_details` is provider-agnostic

The fix lives at `litellm/integrations/arize/_utils.py:247-274`:

```python
prompt_tokens_details = usage.get("prompt_tokens_details") or {}
if hasattr(prompt_tokens_details, "get"):
    cached_tokens = prompt_tokens_details.get("cached_tokens")
    cache_creation_tokens = prompt_tokens_details.get("cache_creation_tokens")
else:
    cached_tokens = getattr(prompt_tokens_details, "cached_tokens", None)
    cache_creation_tokens = getattr(prompt_tokens_details, "cache_creation_tokens", None)
if cached_tokens:
    safe_set_attribute(span, span_attrs.LLM_TOKEN_COUNT_PROMPT_DETAILS_CACHE_READ, cached_tokens)
if cache_creation_tokens:
    safe_set_attribute(span, span_attrs.LLM_TOKEN_COUNT_PROMPT_DETAILS_CACHE_WRITE, cache_creation_tokens)
```

The key observation in the comment is correct and matters for review:

> LiteLLM normalizes provider-specific fields (Anthropic
> `cache_read_input_tokens`, DeepSeek `prompt_cache_hit_tokens`,
> OpenAI native `cached_tokens`) onto `prompt_tokens_details.cached_tokens`,
> so reading from there is provider-agnostic.

This is the right place to read from. The provider transforms already do
the normalization; the OTEL emitter shouldn't re-do it. The previous
deletion (#26506) was exactly the kind of "during a refactor, the call
to `safe_set_attribute(...PROMPT_DETAILS_CACHE_READ...)` got dropped" bug
that the test additions here pin against future regressions.

### Dual-shape support: dict vs Pydantic object

```python
if hasattr(prompt_tokens_details, "get"):
    cached_tokens = prompt_tokens_details.get("cached_tokens")
    ...
else:
    cached_tokens = getattr(prompt_tokens_details, "cached_tokens", None)
```

Handles both the dict shape (when a provider hand-builds the usage
object) and the `PromptTokensDetailsWrapper` Pydantic shape (when
constructed via the typed path). The discriminant is `hasattr(..., "get")`,
which is the right shape for "is this dict-like?". The fall-through
to `getattr(..., None)` correctly defaults to `None` rather than raising
for objects that don't have the attribute.

One subtle correctness point: `usage.get("prompt_tokens_details") or {}`
short-circuits on `None`, so the `hasattr(..., "get")` check in the next
line will always be true on the empty-dict path, which means
`cached_tokens = {}.get("cached_tokens")` → `None`, which then fails
the `if cached_tokens:` truthy gate. Correct end state, but the
short-circuit on `None` is doing real work and is worth a comment so
a future reader doesn't "simplify" it to `usage.get("prompt_tokens_details", {})`
(which behaves identically here but reads differently).

### `if cached_tokens:` vs `if cached_tokens is not None:`

The truthy-check (`if cached_tokens:`) means a value of `0` won't be
emitted. That matches the OpenInference convention: zero cache tokens
are not a meaningful observability signal, and emitting `cache_read = 0`
spans on every uncached call would balloon span attribute counts. So
this is the right gate, and `test_arize_set_attributes_no_cache_tokens_omits_attributes`
at `test_arize_utils.py:438-466` correctly pins this behavior:

```python
attribute_keys = [c.args[0] for c in span.set_attribute.call_args_list]
assert SpanAttributes.LLM_TOKEN_COUNT_PROMPT_DETAILS_CACHE_READ not in attribute_keys
assert SpanAttributes.LLM_TOKEN_COUNT_PROMPT_DETAILS_CACHE_WRITE not in attribute_keys
```

### Test coverage: four targeted cases

`tests/test_litellm/integrations/arize/test_arize_utils.py` adds:

1. `test_arize_set_attributes_anthropic_cache_tokens` — both
   `cache_read_input_tokens=4000` and `cache_creation_input_tokens=261`
   on the Anthropic Usage shape; asserts both `CACHE_READ` and
   `CACHE_WRITE` get set. ✓
2. `test_arize_set_attributes_openai_cached_tokens` — OpenAI's
   `PromptTokensDetailsWrapper(cached_tokens=500)` shape; asserts
   `CACHE_READ` set, `CACHE_WRITE` *not* set (OpenAI native cache has
   no creation concept). ✓ — this is the most important assertion
   because the bug it pins (emitting cache_write=0 for OpenAI) would
   produce misleading dashboards.
3. `test_arize_set_attributes_deepseek_cache_hit_tokens` — DeepSeek's
   `prompt_cache_hit_tokens=1500`; asserts the LiteLLM
   normalization-into-`prompt_tokens_details.cached_tokens` works end-to-end
   and surfaces as `CACHE_READ`. ✓
4. `test_arize_set_attributes_no_cache_tokens_omits_attributes` —
   negative case, no caching at all; asserts neither attribute is
   emitted. ✓

The four cases together pin: every supported provider's cache shape,
the dict-vs-Pydantic discriminant, and the omission contract.

The use of `MagicMock()` for the span and `span.set_attribute.assert_any_call(...)`
is the right test shape — the existing tests in this file use the same
pattern, so it's consistent.

### Comment quality

The block comment at `_utils.py:247-253` is unusually good:

> The OpenInference SpanAttributes constants for these were defined but
> never populated, leaving observability backends (e.g. Langfuse) unable
> to display the cache breakdown.

That's exactly the kind of "why this matters and what was broken
before" context that a future reviewer benefits from. Keeps anyone
from "simplifying" this away later.

## Concerns / nits

1. **Implicit Pydantic-coverage gap**: the test for the Anthropic case
   at `:289` constructs `Usage(... cache_read_input_tokens=4000,
   cache_creation_input_tokens=261)` directly. That goes through the
   `Usage` Pydantic model's auto-normalization (Anthropic-named fields
   → `prompt_tokens_details.cached_tokens` etc.). If anyone ever changes
   the `Usage.__init__` normalization logic, this test will silently
   break the contract this fix relies on. Adding one more test that
   goes "in via the actual Anthropic transform module's output" would
   close that loop, but it's a nice-to-have, not a blocker.

2. **`cache_creation_tokens` is Anthropic-only today.** The code
   correctly reads it as a generic field on `prompt_tokens_details`,
   and OpenInference's `LLM_TOKEN_COUNT_PROMPT_DETAILS_CACHE_WRITE`
   is provider-agnostic — but if a future provider exposes a
   "cache promote" or "cache evict" metric, this code only knows
   about creation. Out of scope; mention in a `TODO`.

3. **No CI status links populated** in the PR description's "CI status
   guideline" section — same nit as is common across LiteLLM PRs. Not
   a code quality issue.

4. The `prompt_tokens_details = usage.get("prompt_tokens_details") or {}`
   line could be a one-liner `prompt_tokens_details = usage.get(...) or {}`
   with a brief comment about why the `or {}` matters when a provider
   sets the field to literal `None`. Minor.

## Verdict reasoning

Surgical observability fix that restores broken behavior (regression
from #26506) and adds a missing-since-day-one feature (cache_write).
Reads from the canonical normalized location (`prompt_tokens_details`),
correctly handles both dict and Pydantic object shapes, uses the
right truthy-gate to suppress zero emissions, and pins all four
relevant scenarios with mock-span tests. Comment in the implementation
documents *why* the fix is at this layer, which protects against
re-regression. This should land as-is.

## What I learned

"Refactor inadvertently dropped a `safe_set_attribute(...)` call" is a
recurring failure mode in observability code: span attribute emissions
are easy to lose in a refactor because they're side-effects with no
return-value coupling. The defense is exactly what this PR does — pin
each emission with a `span.set_attribute.assert_any_call(KEY, VALUE)`
test, and pin the *omission* contract with a "key not in
`call_args_list`" test. The asymmetric test pair (assert-emitted +
assert-not-emitted) is the right shape and worth replicating for
similar OTEL emitter modules in the codebase.
