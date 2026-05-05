# BerriAI/litellm #27177 — feat(batches): request-level Bedrock batch S3 bucket overrides

- Head SHA: `20fcd187b48594cef318f2bad29c02c3833948e0`
- Author: Sameerlite
- Diff: +107 / −0 across 5 files (4 source + 1 new test file)
- Tracker: LIT-2485

## Verdict: `request-changes`

## What it does

Lets a `POST /v1/batches` caller pass `s3_bucket_name` / `s3_output_bucket_name` per request so multi-tenant deployments can route Bedrock batch I/O to per-tenant buckets without standing up a separate deployment alias per tenant. Mechanism:

1. Adds `s3_output_bucket_name` to `GenericLiteLLMParams` (`litellm/types/router.py:248`).
2. Adds both `s3_bucket_name` and `s3_output_bucket_name` to `LiteLLMBatchCreateRequest` typing (`litellm/types/llms/openai.py:436-437`).
3. Proxy endpoint extracts those two fields from the parsed `LiteLLMBatchCreateRequest`, strips, validates, packs them into a private `_litellm_batch_s3_bucket_overrides` dict, and stores it back on `_create_batch_data` (`endpoints.py:124-129`).
4. `create_batch()` reads `_litellm_batch_s3_bucket_overrides` from `kwargs` *after* `litellm_params = dict(GenericLiteLLMParams(**kwargs))`, then writes the override values *into* `litellm_params` so they win over any deployment-level credentials (`main.py:197-202`).
5. Two new tests assert the override path: kwarg-only and override-dict-wins (`tests/test_litellm/proxy/test_batch_s3_bucket_request_overrides.py:1-91`).

## Why I'm holding this for changes

### 1. The proxy populates an undeclared field on a TypedDict (mypy/runtime split)

`endpoints.py:124-129` does:
```python
_create_batch_data = LiteLLMBatchCreateRequest(**data)
_create_batch_data["_litellm_batch_s3_bucket_overrides"] = {
    key: value.strip()
    for key in ("s3_bucket_name", "s3_output_bucket_name")
    for value in [_create_batch_data.get(key)]
    if isinstance(value, str) and value.strip()
}
```

`LiteLLMBatchCreateRequest` is a `TypedDict` (declared at `litellm/types/llms/openai.py:435-437`) and the new `_litellm_batch_s3_bucket_overrides` key is *not* declared on it. Python `TypedDict`s are dicts at runtime so the assignment works, but:
   - mypy / pyright will flag this as `TypedDict ... has no key '_litellm_batch_s3_bucket_overrides'`.
   - The field is conceptually a **private control-plane sentinel** masquerading as user-facing batch-create input. A reviewer reading `LiteLLMBatchCreateRequest` has no way to know this private key exists or what it does.
   - If a downstream consumer ever does `LiteLLMBatchCreateRequest(**_create_batch_data)` (round-trip), the underscore-prefixed key will either be silently dropped or surface as an unknown-field error depending on how strict the constructor is.

Fix: declare the private field on the TypedDict with a leading-underscore convention plus a comment, OR thread the overrides through a separate kwarg into `create_batch()` rather than smuggling them inside `_create_batch_data`. The latter is cleaner — the proxy already passes `**_create_batch_data` to `litellm.acreate_batch()`, so `_litellm_batch_s3_bucket_overrides` hitches a ride through `**kwargs` regardless. A plain `kwargs["_litellm_batch_s3_bucket_overrides"] = {...}` (or pop-based) would avoid the TypedDict pollution.

### 2. The "override dict wins" precedence is *only* enforced when the dict is present, but the dict is *only* populated from the same source fields that `GenericLiteLLMParams(**kwargs)` would already pick up

Walk through: a caller sends `POST /v1/batches` with `s3_output_bucket_name: "tenant-a-out"`.

- `endpoints.py:124-129`: `_create_batch_data["_litellm_batch_s3_bucket_overrides"] = {"s3_output_bucket_name": "tenant-a-out"}`.
- The same `_create_batch_data` (which still contains the *original* `s3_output_bucket_name: "tenant-a-out"` field at the top level) gets passed to `litellm.acreate_batch(**_create_batch_data)`.
- Inside `create_batch()`: `litellm_params = dict(GenericLiteLLMParams(**kwargs))` — this picks up `s3_output_bucket_name="tenant-a-out"` directly.
- Then `litellm_params["s3_output_bucket_name"] = "tenant-a-out"` again from the override dict.

So **for the proxy path the override dict is redundant** — the value would have made it through anyway via the new `GenericLiteLLMParams` field. The override dict only matters when a *programmatic* (non-proxy) caller passes both `s3_output_bucket_name="model-default-output-bucket"` AND `_litellm_batch_s3_bucket_overrides={"s3_output_bucket_name": "request-level-output-bucket"}` (which is exactly what the second test verifies — `test_bedrock_batch_s3_bucket_override_dict_wins_in_litellm_params` at lines 49-91).

That second test exercises a calling pattern that no production code path actually uses — the proxy doesn't construct two competing values. So the test passes but the production proxy path isn't covered. **Add a test that drives the actual proxy endpoint** (e.g. via `app = create_proxy_app(); client.post("/v1/batches", json={...})`) and asserts the bucket landed in the Bedrock URI. Right now the proxy-side branch at `endpoints.py:124-129` has zero direct coverage.

### 3. PR description claims "overrides win over deployment credentials without Bedrock-specific proxy merge logic" — but the override dict is only consulted in `litellm/batches/main.py`, AFTER `GenericLiteLLMParams(**kwargs)` already wins over deployment credentials

Look at the merge order in `main.py:194-202`:
```python
litellm_params = dict(GenericLiteLLMParams(**kwargs))  # request fields populated here
batch_s3_bucket_overrides = kwargs.get("_litellm_batch_s3_bucket_overrides")
if isinstance(batch_s3_bucket_overrides, dict):
    for key in ("s3_bucket_name", "s3_output_bucket_name"):
        value = batch_s3_bucket_overrides.get(key)
        if isinstance(value, str) and value.strip():
            litellm_params[key] = value.strip()
```

The deployment-level credentials are *never read in this function*. They're applied later, inside the Bedrock provider, by merging `litellm_params` over the deployment defaults. So whether the override dict exists or not, the request-level `s3_output_bucket_name` (already in `litellm_params` from `GenericLiteLLMParams`) wins over the deployment default. The override dict isn't doing any independent precedence work.

This means **either the override dict can be removed entirely** (and the change reduces to the `GenericLiteLLMParams` + `LiteLLMBatchCreateRequest` typing additions, ~10 lines) OR the design intent is something else (e.g. routing the override through a path that bypasses `GenericLiteLLMParams` for callers that pass extra kwargs litellm doesn't know about) and that needs to be made explicit in the PR description and test.

### 4. `s3_bucket_name` is in the override dict but no test pins it

The override dict carries both `s3_bucket_name` and `s3_output_bucket_name`, but both new tests only exercise `s3_output_bucket_name`. If `s3_bucket_name` (input bucket) routing has ever-so-slightly different precedence in the Bedrock provider (it usually does — input is read by the batch service, output is written), the test suite won't catch a regression.

### 5. Minor: tuple-iteration trick in the dict-comprehension is hard to read

```python
{
    key: value.strip()
    for key in ("s3_bucket_name", "s3_output_bucket_name")
    for value in [_create_batch_data.get(key)]
    if isinstance(value, str) and value.strip()
}
```

The `for value in [_create_batch_data.get(key)]` walrus-substitute reads as a bug at first glance. A two-line `for` loop or `:=` would be plainer:
```python
overrides: dict[str, str] = {}
for key in ("s3_bucket_name", "s3_output_bucket_name"):
    if isinstance((v := _create_batch_data.get(key)), str) and v.strip():
        overrides[key] = v.strip()
_create_batch_data["_litellm_batch_s3_bucket_overrides"] = overrides
```

Not a blocker on its own but worth fixing while the design is being revisited per (1)/(3).

## What's good

- The new `s3_output_bucket_name` typing on `GenericLiteLLMParams` and `LiteLLMBatchCreateRequest` is the right shape — symmetric with the existing `s3_bucket_name` slot.
- `value.strip()` is applied consistently in both the proxy endpoint and `create_batch()` — no risk of trailing-whitespace bucket names slipping through.
- `isinstance(value, str)` guards against the `None`/non-string cases that a permissive JSON body could deliver.

## Recommended path forward

Either:
- (a) Drop the `_litellm_batch_s3_bucket_overrides` smuggle entirely. The `GenericLiteLLMParams` addition already does the work the PR claims. Net change shrinks to ~10 lines + one test that drives the actual proxy endpoint.
- (b) Keep the override dict but explain the design (e.g. "callers that pass arbitrary kwargs the typed model doesn't know about can still override via this dict") in the PR description and add a test that proves the override dict wins *over* a `GenericLiteLLMParams`-populated value, not just *in addition to* it.

Either way, declare the private key on the TypedDict (or stop putting it there) and add real proxy-endpoint coverage.

## Citations

- `litellm/proxy/batches_endpoints/endpoints.py:124-129` — proxy populates `_litellm_batch_s3_bucket_overrides`
- `litellm/batches/main.py:194-202` — `create_batch` merges overrides after `GenericLiteLLMParams`
- `litellm/types/router.py:248` — `s3_output_bucket_name` added to `GenericLiteLLMParams`
- `litellm/types/llms/openai.py:435-437` — bucket fields added to `LiteLLMBatchCreateRequest`
- `tests/test_litellm/proxy/test_batch_s3_bucket_request_overrides.py:1-91` — two tests, neither drives the proxy endpoint
