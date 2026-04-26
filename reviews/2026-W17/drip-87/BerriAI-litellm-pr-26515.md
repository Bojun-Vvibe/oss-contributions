# Review — BerriAI/litellm#26515: fix(ui): render dict-shaped fallback entries in router settings

- **Repo:** BerriAI/litellm
- **PR:** [#26515](https://github.com/BerriAI/litellm/pull/26515)
- **Author:** jayy-77 (Jay Prajapati)
- **Head SHA:** `1efcbceed72e62c581630fbb8ae0fac95a8e85d4`
- **Size:** +263 / −60 across 5 files
- **Verdict:** `merge-after-nits`

## Summary

Fixes issue #26473: the Admin UI's Router Settings → Fallbacks tab crashes with React error #31 ("Objects are not valid as a React child") when `router_settings.fallbacks` contains a dict-shaped chain entry like `{model: "fallback-model", api_key: "...", extra_headers: {}}`. The runtime backend supported this shape (per `litellm/router_utils/fallback_event_handlers.py:122-132`'s `kwargs.update(mg)` for dict items) but the UI typed each chain entry as `string` and rendered it directly. PR widens the type to `FallbackChainItem = string | { model?: string; [key: string]: any }`, adds `getFallbackChainItemDisplayName()`, tags dict entries with an `override` badge, and masks secrets (`api_key`, `authorization`, `token`, `secret`, `password` and matching keys inside `extra_headers`) before they can enter the DOM.

## Technical assessment

The render-crash diagnosis is exactly right. `Objects are not valid as a React child` is React's defense against passing a raw object where a `ReactNode` is expected; the fix has to either (a) string-coerce the object before render or (b) destructure into typed children. This PR does (b) for the display name and (a) for the JSON-tooltip body, both of which are correct.

The masking surface is the most important nit-magnet. From the body, the masked keys are `api_key`, `authorization`, `token`, `secret`, `password`, plus matching keys inside `extra_headers`. That's a reasonable starter set but case-handling matters: `Authorization` (capital A) is the canonical HTTP header form, `X-Api-Key` is common, `Bearer` tokens often live under `Authorization`. The implementation should be case-insensitive on key lookup. The PR body says masking is by key name match — assuming the implementation uses `key.toLowerCase()` against a lowercased denylist, this is fine; if it uses literal-case match, `Authorization` leaks.

Test coverage is proportional and surgical: 5 new test cases covering (1) no-crash render with dict entry, (2) override-tag rendering, (3) mixed string+dict in same chain (exactly one override tag), (4) round-trip safety on sibling delete, (5) `api_key` not in DOM. Test 5 is the right form of the masking guarantee — assert against the rendered DOM rather than against intermediate state, because DOM is what an XSS-curious user would inspect.

The unrelated Python diffs (`factory.py`, `predibase/chat/transformation.py`, `xecguard.py`) are all pure formatting changes — line-length wrapping, trailing newline. These look like an `autopep8` / `ruff format` pass landed alongside the UI fix. Three files, ~20 lines, zero behavior change.

## Nits worth addressing pre-merge

1. **Strip the formatting-only Python diffs into a separate PR.** `litellm/litellm_core_utils/prompt_templates/factory.py:5294`, `litellm/llms/predibase/chat/transformation.py:175-210`, and `litellm/proxy/guardrails/guardrail_hooks/xecguard/xecguard.py:215-220` are pure reflow/format changes with no relation to the UI bug. Mixing them with a UI fix PR makes review noisier than it has to be and introduces merge-conflict surface unrelated to the actual change. Easy ask: revert them here, ship as a separate `chore: ruff format` PR.

2. **Confirm masking is case-insensitive.** From the diff window I can't see the exact implementation of the masking predicate. Verify that `Authorization`, `API_KEY`, `X-Api-Key` all get masked, not just literal-lowercase `api_key`. Add a test case asserting `Authorization: Bearer sk-...` is rendered as `Authorization: ••••••`, not the literal token.

3. **Override tooltip with masked-JSON.** The body says hovering the override badge shows masked JSON in a tooltip. Tooltips that contain user-supplied data should also be HTML-escaped — confirm React's default text-rendering is being used (it is by default), not `dangerouslySetInnerHTML`. Quick grep.

4. **`[key: string]: any`.** The `FallbackChainItem` type at the UI layer uses `any` for unknown keys, which is the path of least resistance but loses type safety for known override keys (`api_key`, `extra_headers`, `extra_body`, `timeout`, `mock_response`). Consider a discriminated union of "well-known override keys with typed values" + `[key: string]: unknown` as the escape hatch. Pure code-hygiene nit; not blocking.

5. **`getFallbackChainItemDisplayName()` fallback to stringified JSON.** The body says when `dict.model` is missing, the function falls back to `JSON.stringify(item)`. That stringified form will include unmasked secrets if no `model` key is present. Either run the masking pass first then JSON-stringify, or refuse to render and show a "malformed override" placeholder. Real masking-bypass risk worth a dedicated test.

6. **React key stability.** The body says keys are `${i}-${displayName}`. `i + displayName` is fine for stable rendering within one list, but if the same model appears with two different overrides (`gpt-4` with `api_key=A` and `gpt-4` with `api_key=B`), both produce the same key. Use the override JSON (masked) or a hash of the entry as the key tail.

## Verdict rationale

`merge-after-nits`. The bug is real (production crash on a documented config shape), the fix is correctly scoped to the UI render layer with no backend coupling, test coverage is exemplary for a UI bug fix (5 surgical cases including a regression-locking test for the React error). Address items 1 (split formatting churn), 2 (case-insensitive masking test), and 5 (JSON.stringify masking-bypass) before merge. Items 3, 4, 6 can be follow-ups.
