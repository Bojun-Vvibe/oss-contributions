---
pr: 4726
repo: browser-use/browser-use
sha: 1a6a5e459f1c5b5ac1ee8bc7b3d0efd0f5132102
verdict: request-changes
date: 2026-04-25
---

# browser-use/browser-use#4726 — feat: add Astraflow provider support

- **URL**: https://github.com/browser-use/browser-use/pull/4726
- **Author**: ucloudnb666

## Summary

Adds Astraflow (UCloud / 优刻得 — an OpenAI-compatible model
aggregator) as a first-class provider. Mirrors the layout of existing
providers like `openrouter` and `deepseek`: new `browser_use/llm/
astraflow/{chat.py, serializer.py}`, two env-var hooks in
`browser_use/config.py`, lazy-import wiring in `browser_use/llm/
__init__.py`, and a model catalog entry in `browser_use/llm/
models.py`. Net: +274/-1 across 6 files.

## Reviewable points

- **Two endpoints, two env vars, one client default**: `chat.py:38`
  hard-codes `base_url = 'https://api-us-ca.umodelverse.ai/v1'` (the
  global endpoint, keyed by `ASTRAFLOW_API_KEY`). The China endpoint
  (`https://api.modelverse.cn/v1`, keyed by `ASTRAFLOW_CN_API_KEY`)
  exists in `.env.example` and `config.py:157-163` but is **never
  wired up automatically** — users have to pass `base_url=` manually
  to switch. That's confusing because the env var hints at automatic
  region routing. Either:
  (a) document that `ASTRAFLOW_CN_API_KEY` requires explicitly
      constructing `ChatAstraflow(api_key=…, base_url=…)`, or
  (b) add a `region: Literal["global","cn"] = "global"` field to the
      dataclass that picks `base_url` and `api_key` accordingly.
  Option (b) matches the implicit promise of two env vars.

- **`_get_usage` ignores reasoning-token details**
  (`chat.py:_get_usage` around line 172): it reads
  `prompt_tokens_details.cached_tokens` but never looks at
  `completion_tokens_details.reasoning_tokens`, even though Astraflow
  proxies Claude/GPT/Gemini — i.e. the upstream models *do* return
  reasoning tokens. The other providers in this repo (e.g.
  `ChatOpenAI._get_usage`) record reasoning tokens. Consistency bug.

- **Structured-output path raises `ModelProviderError` with
  `status_code=500` on `content is None`** (around the structured
  branch in `ainvoke`). 500 implies "Astraflow's server failed", but
  the model returning `None` content under a strict JSON schema is
  much more likely a model-side schema-non-compliance issue and
  should map to a 4xx-like signal so client retry policies don't loop.
  Compare with `ChatOpenRouter` which uses
  `ModelProviderError(...)` without status_code or with 502/503.

- **No tests at all**. Other provider PRs in this repo (e.g.
  `tests/llm/test_openai.py`) ship at least a
  serializer round-trip test plus a mocked `chat.completions.create`
  test. The Astraflow PR is 274 lines of new provider code with
  zero coverage. This is the strongest blocker — the bare minimum
  should be a test that exercises `AstraflowMessageSerializer.
  serialize_messages` against a representative `BaseMessage` set, plus
  a mocked-`AsyncOpenAI` test for `ainvoke` with and without
  `output_format`.

- **`max_retries: int = 10`** (chat.py:37) is unusually high — most
  providers in this repo default to 0–3. With Astraflow being an
  aggregator that may itself retry upstream, multiplying retries can
  produce O(retries × upstream_retries) attempts and large bills on
  rate-limit storms. Recommend dropping to 3 (or the default of
  whatever sibling providers use) unless there's a specific reason.

- **`models.py`** gets +26/-1 — confirming the model catalog
  additions reference real Astraflow model IDs and not aspirational
  ones is best left to the maintainer / author, but worth a sanity
  pass.

- **No banned strings; no secrets shipped; env vars named clearly.**
  Clean from a privacy/security standpoint.

## Rationale

`request-changes`. The shape is right and the file layout matches
sibling providers, but the missing tests, the unused-CN-endpoint
config UX, the default `max_retries=10`, and the missing reasoning-
tokens accounting are all things that would be cheap to fix and that
users will hit immediately. Once tests land and the dual-endpoint
wiring is either documented or automated, this becomes
`merge-after-nits`.

## What I learned

For provider PRs to a multi-provider library, the litmus test isn't
"does the new provider work" — it's "does the new provider behave
*identically* to siblings on the cross-cutting concerns" (retries,
usage accounting, error mapping, env-var conventions). A new provider
with a 10× retry default or with bespoke error codes silently breaks
user retry budgets and observability dashboards.
