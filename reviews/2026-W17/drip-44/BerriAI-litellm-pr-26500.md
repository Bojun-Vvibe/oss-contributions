# BerriAI/litellm PR #26500 — Wrap extra_body for JSON-configured OpenAI-compatible providers

- **URL:** https://github.com/BerriAI/litellm/pull/26500
- **Author:** @jayy-77
- **State:** OPEN (base `main`)
- **Head SHA:** `2fc08fa`
- **Size:** +84 / -3
- **Files:** `litellm/utils.py`, `tests/test_litellm/test_utils.py`
- **Closes:** #26443

## Summary of change

`add_provider_specific_params_to_optional_params()` decides whether to
wrap unknown kwargs (e.g., `chat_template_kwargs`) inside `extra_body`
or leave them at the top level. The "wrap" branch was previously
gated only on `custom_llm_provider in ["openai", "azure",
"text-completion-openai"] + litellm.openai_compatible_providers`.

JSON-registered providers (declared in `providers.json` rather than
hard-coded in `openai_compatible_providers`) — the issue cites
Scaleway — also route through the OpenAI client path in `main.py`,
but missed this branch and had unknown params promoted to top-level
kwargs, where the OpenAI SDK rejected them with "unexpected keyword
argument".

The fix adds `JSONProviderRegistry.exists(custom_llm_provider)` as a
second OR clause to the gate.

## Findings against the diff

- **`litellm/utils.py` L4760–4768**: the new disjunction is correct
  shape — same wrapping behavior is now applied to JSON-registered
  providers without adding them to the static
  `openai_compatible_providers` list. This is the right factoring;
  the static list and the JSON registry are two surfaces of the same
  "OpenAI-client-routed" trait, and the wrap-for-extra_body logic
  should observe both.
- **Import at L390**: `from litellm.llms.openai_like.json_loader import
  JSONProviderRegistry` is added at module scope. `utils.py` is hot —
  worth confirming this doesn't introduce a circular import on cold
  start (the loader presumably imports nothing from `utils` itself,
  but a lazy import inside the gate would be a safer pattern if there's
  any doubt).
- **Test 1 (`test_json_provider_wraps_unknown_param_in_extra_body`)**:
  good shape — asserts `JSONProviderRegistry.exists("scaleway")` and
  `"scaleway" not in litellm.openai_compatible_providers` as
  preconditions, so the test will fail loudly if the registry is
  reorganized rather than silently passing on the wrong axis.
- **Test 2 (`test_native_non_openai_provider_still_top_level`)**:
  uses `bedrock` as the negative control, asserting unknown params
  *stay* at top level. This is the right guard against the fix
  over-broadening — "wrap everything in extra_body" would silently
  break native non-OpenAI handlers that consume those params directly.
- **Comment quality**: the inline comment ("JSON-configured providers
  route through the OpenAI client (see main.py routing branch), so
  they need the same extra_body wrapping…") points at the routing
  invariant that makes the change valid. Future maintainers will
  thank the author.

## Verdict

**merge-after-nits**

The fix is correct and the tests pin the right invariants. Nits:

1. Consider lazily importing `JSONProviderRegistry` inside the function
   if there's any risk of import cycles in `utils.py`. Cheap insurance.
2. The two-clause `or` would read more cleanly as a small helper
   `_is_openai_client_routed(provider)` that returns the disjunction —
   future provider-routing additions go in one place, and the gate
   stays a one-liner. Optional.
3. Worth adding a third test pinning that a provider in
   `openai_compatible_providers` *and* in the JSON registry doesn't
   accidentally double-wrap (defensive, but `extra_body` merge
   semantics are the kind of thing that breaks once and never gets
   noticed until production).

## What I learned

When two registration surfaces (a static Python list and a JSON-loaded
registry) describe the same trait, every gate that reads only one of
them is a latent bug. The fix here is a one-line `or`, but the durable
fix is to extract the trait into a single predicate — the cost of
"add new clause whenever someone notices another miss" compounds.
This is the same anti-pattern as having both `feature_flags.is_on()`
and `os.environ.get("FEATURE_…")` checked in different code paths
for what is logically one feature.
