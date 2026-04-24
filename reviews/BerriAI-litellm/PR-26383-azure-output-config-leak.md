# PR-26383 — fix: prevent Azure output_config leakage

[BerriAI/litellm#26383](https://github.com/BerriAI/litellm/pull/26383)

## Context

One-line fix in
`litellm/llms/anthropic/experimental_pass_through/adapters/handler.py`:
`_prepare_completion_kwargs` was building the kwargs dict that gets
forwarded to the underlying completion call (in this code path, an
Azure deployment). The function already excluded `anthropic_messages`
from being passed through. The patch widens that exclusion set to
`{"anthropic_messages", "output_config"}`.

Why: when a caller passes `thinking={"type": "adaptive"}` plus
`extra_kwargs={"output_config": {"effort": "high"}}`, the adapter
*translates* `output_config.effort` into `reasoning_effort="high"` on
the kwargs going to the upstream provider. But it was *also* leaving
`output_config` in the dict — Azure (and most non-Anthropic backends)
doesn't recognize the `output_config` key and either ignores it,
422s, or worse, the field gets serialized into the request body where
some Azure deployments will reject the entire request.

The new test
(`test_messages_handler_does_not_forward_output_config_to_azure`)
calls `_prepare_completion_kwargs` directly with `model="azure/gpt-5.2-
chat"`, `thinking={"type":"adaptive"}`, and `extra_kwargs` containing
`output_config`. It asserts:

1. `completion_kwargs["reasoning_effort"] == "high"` (translation
   happened),
2. `completion_kwargs["api_version"] == "2025-01-01-preview"` (other
   `extra_kwargs` still pass through),
3. `"output_config" not in completion_kwargs` (the leak is closed).

## Why it matters

`output_config` is an Anthropic-Messages-API concept used by the
adaptive-thinking surface. Forwarding it to Azure is a clean
abstraction violation — and the worst kind, because the symptom
depends on the Azure deployment's strict-schema setting. Some
deployments tolerate unknown fields (silent waste of payload
bandwidth); some return cryptic 400s; some pass it through to a model
that treats it as user-controlled data. The unification of the
exclusion set is correct.

## Strengths

- Genuinely one-line behavior change; everything else is test scaffolding.
- The test sites the assertion at the right boundary: the *translation*
  (`output_config.effort` → `reasoning_effort`) still works, only the
  raw key is dropped. That's the contract.
- Asserting on three orthogonal properties (translation present, pass-
  through other extras, leak closed) catches the three failure modes
  this fix could have introduced if done wrong.
- `_prepare_completion_kwargs` is tested directly rather than through
  the full handler, which keeps the test fast and lets it run in any
  CI lane that doesn't have Azure credentials.

## Concerns / risks

- Hardcoded set literal `{"anthropic_messages", "output_config"}`. The
  list of "Anthropic-only adapter-internal kwargs that shouldn't leak"
  is going to grow (e.g. `tool_choice` shapes, `system` prompt
  formats, future thinking variants). A constant
  `_ADAPTER_INTERNAL_KEYS` at module scope would scale better and
  centralize the policy.
- The test asserts the leak is closed *for Azure*, but the exclusion
  is unconditional. That's correct (the Anthropic native path doesn't
  go through this `_prepare_completion_kwargs`), but the test name and
  comment frame it as Azure-specific, which could mislead a future
  reader into thinking the exclusion is route-conditional.
- No test for the case where `output_config` is the *only* relevant
  extra and `thinking` is *not* `adaptive` — i.e. translation doesn't
  fire. Does `output_config` still get dropped silently, or should it
  surface a warning? The patch drops it silently in all cases; that
  may surprise users who expected `output_config` to be honored
  through pass-through.
- The exclusion set is matched by exact key only. If the caller passes
  `Output_Config` (typo'd casing) it won't be excluded. Probably fine
  given Pydantic models elsewhere normalize, but worth knowing.

## Suggested follow-ups

- Promote `excluded_keys` to a module-level `frozenset`
  `_ADAPTER_INTERNAL_KEYS` with a docstring listing why each key is
  there and which translation consumes it.
- Add a counterpart test for the Responses adapter
  (`responses_adapters/transformation.py`) — the existing
  `test_responses_adapter_adaptive_with_output_config` two functions
  below suggests there's a parallel surface; check for the same leak
  there.
- Consider emitting a `verbose_logger.debug("dropping adapter-internal
  key %s", key)` so users debugging "where did my output_config go"
  can find it in the logs.
- Add a regression test for `output_config` *without* adaptive thinking
  to lock in the "always drop" semantics.
