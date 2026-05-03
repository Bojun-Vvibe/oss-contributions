# BerriAI/litellm PR #26980 — fix(gemini): avoid duplicate model route for full api_base

- Link: https://github.com/BerriAI/litellm/pull/26980
- SHA: `f981e4abb0f4890e5cef4963b711e7511bf458af`
- Author: StatPan
- Stats: +79 / −7, 4 files

## Summary

Fixes a Gemini custom-`api_base` URL construction bug where, if the operator already provided a full `…/models/{model}:{endpoint}` URL (or a `…/models/{model}` URL), litellm would unconditionally append `/models/{model}:{endpoint}` again, producing a malformed double-routed URL. Replaces the unconditional concat with three branches keyed on the suffix of `api_base`. Also tightens the tag-based routing strict-tag check and adds parametrised regression tests.

## Specific references

- `litellm/llms/vertex_ai/vertex_llm_base.py` L412–L422: replaces the single `url = "{}/models/{}:{}".format(api_base, model, endpoint)` with three cases (`endswith(/models/{model}:{endpoint})`, `endswith(/models/{model})`, fallback). The fallback now also `rstrip("/")`s the base, fixing a separate latent issue where a trailing slash produced `…//models/…`. Worth calling out explicitly in the PR body since it is a behaviour change beyond the stated fix.
- `litellm/llms/vertex_ai/vertex_llm_base.py` L420 (the `endswith(f"/models/{model}")` branch): if `model` contains regex-significant characters or path separators (e.g. a fine-tuned model id with a slash), `endswith` is still safe (string compare) but the resulting URL might be unexpected. Probably fine for current Gemini model id space.
- `litellm/router_strategy/tag_based_routing.py` L106–L110: changes `bool(deployment_tags)` to an explicit `deployment_tags is not None and len(deployment_tags) > 0`. Equivalent for lists, but distinguishes `None` (no policy) from `[]` (empty policy). Good defensive readability change; verify no caller is currently relying on `[]` and `None` being collapsed.
- `tests/test_litellm/llms/vertex_ai/test_vertex_llm_base.py` L891–L928: new `@pytest.mark.parametrize` covering the full-route, model-prefix, and trailing-slash cases. The three parametrised cases map 1:1 to the three new branches in `_check_custom_proxy`. Coverage looks complete.
- Same test file L71: `print(f"result: {result}")` is replaced with `assert result == ("fake-token-1", "different-project")` — drive-by test improvement, separate from the main fix; harmless but worth flagging in the description.

## Verdict

verdict: merge-as-is

## Reasoning

Narrow, well-tested fix for a real and previously-reported issue (#24786). The branching is exhaustive over the observed `api_base` shapes, the `rstrip("/")` quietly fixes a related trailing-slash bug, and the parametrised tests cover all three new code paths. The drive-by test cleanup and tag-routing readability change are both improvements. No blockers.
