# BerriAI/litellm#26674 — feat: add Astraflow (ModelVerse) provider support

- **Repo**: BerriAI/litellm
- **PR**: [#26674](https://github.com/BerriAI/litellm/pull/26674)
- **Head SHA**: `ee9256ac5afbd84ed3e5b61cb1a3001336a22ddc`
- **Reviewed**: 2026-04-29
- **Verdict**: `merge-after-nits`

## Context

Adds two new OpenAI-compatible provider entries for Astraflow (UCloud
ModelVerse) — a model aggregation platform. Two providers, one per
region: `astraflow` (US/CA, base `https://api-us-ca.umodelverse.ai/v1`)
and `astraflow-cn` (China, base `https://api.modelverse.cn/v1`), each
with their own env-var-keyed API key.

This is the standard "JSON-configured OpenAI-compatible provider"
shape that litellm has codified into a 3-file pattern.

## What changed (3 files, 4 hunks)

### `litellm/llms/openai_like/providers.json`

```json
"astraflow": {
  "base_url": "https://api-us-ca.umodelverse.ai/v1",
  "api_key_env": "ASTRAFLOW_API_KEY"
},
"astraflow-cn": {
  "base_url": "https://api.modelverse.cn/v1",
  "api_key_env": "ASTRAFLOW_CN_API_KEY"
}
```

Inserted alphabetically between `assemblyai` and `charity_engine`.
Standard JSON entries; no provider-specific quirks.

### `litellm/types/utils.py`

```python
class LlmProviders(str, Enum):
    ...
    ASTRAFLOW = "astraflow"
    ASTRAFLOW_CN = "astraflow-cn"
```

Added at the end of the enum (separated by a blank line from the
`BEDROCK_MANTLE` entry above — minor style nit, see below).

### `litellm/constants.py`

Two list additions:

```python
# openai_compatible_endpoints (URLs without protocol)
"api-us-ca.umodelverse.ai/v1",
"api.modelverse.cn/v1",

# openai_compatible_providers (provider slugs)
"astraflow",         # Astraflow (ModelVerse) Global node
"astraflow-cn",      # Astraflow (ModelVerse) China node
```

The endpoint list (used for auto-detection from `api_base`) entries
are written *without* the `https://` prefix, while
`providers.json` includes the prefix. That's consistent with how
the existing `openai_compatible_endpoints` list is structured, but
worth flagging as a footgun for future contributors who don't
inspect the existing entries first.

## Things I'd want changed before merge

1. **Slug uses dash, enum uses underscore**. The `LlmProviders` enum
   member is `ASTRAFLOW_CN` mapped to value `"astraflow-cn"`. The
   slug already exists as the dash form everywhere else in the
   diff, so the enum *value* is right. Just confirm the codebase
   doesn't anywhere convert `LlmProviders.ASTRAFLOW_CN.name.lower()`
   into a slug — if it does, you get `astraflow_cn` (underscore)
   and a silent mismatch with the JSON registry. The other enum
   members in the file (`SAMBANOVA`, `BEDROCK_MANTLE`) don't have
   this hyphen-vs-underscore split so the precedent isn't strong.
   Quick grep for `.name.lower()` and `.value` usage at the call
   sites that resolve a provider would resolve this in 30 seconds.

2. **No model entries / pricing**. The PR adds the *provider* but
   no `model_prices_and_context_window.json` entries. Users will
   have to spell out `astraflow/<model_name>` and litellm will
   route via the OpenAI-compatible client without any
   per-model context-window or cost data. That's an acceptable
   first cut — mirrors how `featherless_ai`, `nebius`, and
   similar aggregator providers landed initially — but worth
   calling out in the README/docs hunk (which doesn't seem to
   exist in this diff). At minimum: a comment in `providers.json`
   noting "users supply model after provider slug; pricing not
   yet wired".

3. **Endpoint protocol stripping**. Adding endpoints without the
   protocol to `openai_compatible_endpoints` (presumably for prefix
   matching against `api_base` strings) is the existing pattern,
   but a small helper (`_endpoint_host(url) -> str`) reused at the
   add-site would prevent the next contributor from copying the
   wrong shape. Out of scope for this PR.

4. **No tests in the visible diff hunks**. The PR description
   shows a usage code snippet but no `tests/test_litellm/...`
   files in the diff I'm seeing. The other "feat: add X provider"
   PRs in litellm typically add at least one
   `test_openai_like_providers.py` smoke test that asserts the
   provider is registered, the endpoint resolves, and the API
   key env var is read. I'd want one of those before merge.

5. **PR template box** ("I have Added testing in
   tests/test_litellm/") should be checked or explicitly waived
   in the description.

## Risk analysis

- **Backwards-compat**: pure additive. Existing providers and
  routing logic untouched.
- **Routing collision**: neither slug collides with an existing
  entry in the providers list. Endpoint base URLs are unique to
  Astraflow's domains, so auto-detection by `api_base` won't
  misroute.
- **Region splitting**: registering two providers (instead of one
  with a region param) matches how `azure`/`vertex` are *not*
  done but is consistent with how `openai-compatible` aggregators
  generally are done in this codebase. Defensible.
- **OpenAI-compat assumption**: provider says "OpenAI-compatible"
  in the PR body but not in the diff. If the actual API has
  quirks (different streaming format, different tool-call schema)
  this PR would silently fail at first request. A request-shape
  smoke test would catch this — see point 4 above.

## Verdict

`merge-after-nits`. The provider-registration shape is correct and
follows the established 3-file pattern, but I'd want a smoke test
checked in and the enum-name-vs-slug hyphen point either confirmed
non-issue or fixed before merge. Neither is large; both are
worth doing now rather than as follow-ups.

## What I learned

For LLM proxy/router projects, "add a new OpenAI-compatible provider"
PRs are mechanical *but* there's a long tail of subtle conventions
(URL with-vs-without protocol in different lists,
slug-vs-enum-name-vs-value, alphabetical insertion order) that
project maintainers care about because the lists are growing and
will eventually be ground truth for downstream tooling. PRs that
mirror the existing entries field-by-field, alphabetically, with
matching comments, get reviewed faster than ones that "look about
right".
