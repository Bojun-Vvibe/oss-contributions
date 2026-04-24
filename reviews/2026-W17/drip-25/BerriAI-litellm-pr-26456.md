# BerriAI/litellm PR #26456 — fix(openai): gpt-5.5 does not support reasoning_effort=minimal

- **Repo:** BerriAI/litellm
- **PR:** [#26456](https://github.com/BerriAI/litellm/pull/26456)
- **Head SHA:** `34c93645e9aaed8546032ef52ff71b54b264e454`
- **Author:** mateo-berri (Mateo Wang)
- **Size:** +48 / −8 across 3 files
- **Reviewer:** Bojun (drip-25)

## Summary

Fixes a JSON-metadata bug introduced in #26449 (the gpt-5.5 model
catalog addition that landed earlier this drip cycle). All four
of the new entries — `gpt-5.5`, `gpt-5.5-2026-04-23`,
`gpt-5.5-pro`, `gpt-5.5-pro-2026-04-23` — were marked
`supports_minimal_reasoning_effort: true`. Live calls against
OpenAI's Chat Completions API on 2026-04-24 confirm both models
return HTTP 400:

```
"Unsupported value: 'reasoning_effort' does not support 'minimal' with this model.
 Supported values are: 'none', 'low', 'medium', 'high', and 'xhigh'."
```

With the flag wrong, LiteLLM was silently round-tripping
`minimal` to OpenAI and surfacing a 400 to the caller. After this
fix `OpenAIGPT5Config._is_reasoning_effort_level_explicitly_disabled()`
returns `True`, so LiteLLM either drops the param (when
`drop_params=True`) or raises a local `UnsupportedParamsError`
*before* the API call.

## Key changes

### `model_prices_and_context_window.json` and `litellm/model_prices_and_context_window_backup.json` (+4 / −4 each)

Flips `supports_minimal_reasoning_effort: true → false` on all
four entries. The dual-file edit is correct (litellm carries the
backup copy in-tree to survive cold-starts where the live JSON
fetch fails) — both files must agree.

### `tests/test_litellm/litellm_core_utils/llm_cost_calc/test_llm_cost_calc_utils.py` (+40 / −0)

New parametrized regression test
`test_gpt55_reasoning_effort_flags_match_live_openai_api` that
pins `supports_{none,minimal,xhigh}_reasoning_effort` on each of
the four entries to OpenAI's actual API contract. Test rows:

```
[gpt-5.5-True-True-False]
[gpt-5.5-2026-04-23-True-True-False]
[gpt-5.5-pro-False-True-False]
[gpt-5.5-pro-2026-04-23-False-True-False]
```

Note the asymmetry: `gpt-5.5` and `gpt-5.5-2026-04-23` accept
`none`, the `-pro` variants don't. The test parametrisation
captures that correctly.

## Concerns

1. **`gpt-5.5-pro` also rejects `reasoning_effort="low"`, but the
   schema can't represent that today.**
   The PR body explicitly flags this as out of scope and worth a
   follow-up:
   > `gpt-5.5-pro` additionally rejects `reasoning_effort="low"`
   > (only accepts `medium, high, xhigh`). The current JSON schema
   > has no `supports_low_reasoning_effort` flag, so this isn't
   > representable without a small schema + transformation
   > extension. Filing separately.

   Operationally this means a user passing `reasoning_effort="low"`
   to `gpt-5.5-pro` through LiteLLM will still hit a silent 400
   round-trip after this PR lands. Worth landing the schema
   extension follow-up before any docs / changelog point users at
   `gpt-5.5-pro` for low-effort work.

2. **Dual-JSON-file edit needs an in-CI consistency check.**
   `model_prices_and_context_window.json` and
   `litellm/model_prices_and_context_window_backup.json` are
   supposed to stay byte-equivalent for the model-capability
   columns. This PR edits both correctly, but the project
   doesn't appear to have a CI guard that asserts the two files
   stay in sync for shared keys. A tiny pytest that diffs the
   two JSONs on every shared model entry would prevent the next
   maintainer from updating only one and silently regressing
   cold-start behaviour.

3. **Test verifies the flags, but not the runtime path.**
   The new regression test asserts the JSON values. It doesn't
   exercise the actual LiteLLM call path — i.e. "when a client
   sends `reasoning_effort=minimal` to `gpt-5.5` with
   `drop_params=True`, the param is dropped and the call goes
   through; without `drop_params`, an `UnsupportedParamsError`
   is raised locally". Worth adding an integration-style test
   under `tests/test_litellm/llms/openai/` that mocks the
   OpenAI client and asserts both branches. Otherwise a future
   change to `_is_reasoning_effort_level_explicitly_disabled()`
   could regress the runtime path while the JSON test still
   passes.

4. **The "verified live against OpenAI" claim is the load-bearing fact.**
   The PR body shows curl output proving 400s before the fix
   and 200s after with `drop_params=True`. That's the correct
   evidence. Worth pinning the verification date (2026-04-24)
   and the `gpt-5.5-pro` model snapshot in a comment near the
   parametrize block — model behaviour can shift; the
   regression test asserts a *specific snapshot* of OpenAI's
   contract.

## Verdict

`merge-as-is` — three-file fix with a parametrized regression
test, evidence from live API calls, an honest scope-limit on
the `low` schema gap. The dual-JSON edit is correct, the test
captures the asymmetry between `-pro` and non-`-pro` variants.
Two follow-ups worth filing as issues but not blocking: the
`supports_low_reasoning_effort` schema extension (so the
`-pro` `low` rejection becomes representable), and a JSON
consistency-check test for the two model-prices files.

## What I learned

"Capability flags in static JSON metadata" is a brittle pattern
because the source of truth (the upstream API) can drift, and
the JSON has no enforcement that it matches reality. The
correct defensive shape is what this PR did: *every* capability
flag claim should be paired with a regression test that calls
out the API behaviour it asserts, and ideally a date-stamped
comment so the next maintainer knows when the claim was last
verified. Without that, capability-flag PRs accumulate as
"vendor said it works, didn't actually test, surfaced as 400 to
a downstream user three weeks later" — exactly the failure
mode this PR is fixing. Same shape applies to model-context-
window fields, max-output-token fields, and supports-tools
fields. Treat the capability JSON as a public contract and
test it like one.
