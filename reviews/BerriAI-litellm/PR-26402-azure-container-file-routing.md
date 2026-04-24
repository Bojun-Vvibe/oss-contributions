# PR #26402 — route Azure container file requests by decoded deployment

**Repo:** BerriAI/litellm • **Author:** Sameerlite • **Status:**
open • **Net:** +201 / −23

## Change

Container file endpoints (`/v1/containers/{cid}/files/...`) on the
proxy were decoding the managed `container_id` to recover the
original provider-side ID, but were *not* picking up the encoded
`model_id` to resolve the originating Azure deployment's
credentials (api_base / api_key / api_version). Result: file
requests targeted at an Azure deployment ended up reconstructing
URLs like `/openai/responses/openai/containers/...` or hitting the
wrong region.

This PR factors the decode logic into
`_decode_container_routing_metadata` and adds
`_get_deployment_credentials_for_model_id`, which pulls deployment
creds from the global `llm_router` (with a `model_group_name`
fallback). The three handler entry points
(`_process_binary_request`, `_process_multipart_upload_request`,
`_process_request`) now call the shared decode helper and, when a
`model_id` is recovered, override `litellm_params.api_base`,
`api_key`, and `api_version`. Two regression tests added.

## Load-bearing risk

1. **Global `llm_router` import inside the helper.** The new
   `_get_deployment_credentials_for_model_id` does
   `from litellm.proxy.proxy_server import llm_router` *inside*
   the function. This is the established proxy pattern (avoids
   import cycles), but it makes this code path untestable in
   isolation without monkeypatching that module attribute. The
   added test (`test_regression_binary_file_request_uses_
   deployment_api_base`) presumably does that monkeypatch
   (truncated in the diff sample), but new contributors will
   trip on this every time.

2. **Silent fallback to `model_group_name`.** When
   `get_deployment_credentials(model_id=...)` returns falsy, the
   helper falls back to
   `llm_router.get_deployment_by_model_group_name(model_group_
   name=str(model_id))`. The encoded `model_id` is supposed to be
   a deployment ID (e.g. `model_abc123`), not a model-group name.
   This fallback will happily resolve a *different* deployment if
   a model-group with the same string name exists — silently
   re-routing the request. Should at minimum log a warning when
   the fallback fires.

3. **Credential override is unconditional once `model_id` is
   present.** The diff overrides `api_base`/`api_key`/
   `api_version` from the deployment creds even if the caller
   passed explicit values via `litellm_params`. For multi-tenant
   proxies where caller-supplied keys are intentional (BYOK), this
   silently swaps in the proxy-managed deployment credentials.
   Should be a *fill-defaults*, not an *overwrite*: only set
   each key if the caller didn't provide it.

4. **`data["model_id"] = decoded_model_id` in
   `_process_multipart_upload_request` and `_process_request`**
   leaks the decoded model_id into the request payload. Any
   downstream code that round-trips `data` to logs (spend logs,
   audit trail) now persists the deployment's internal ID, which
   may be sensitive in shared deployments. Confirm the spend-log
   serializer redacts this or that the field is intended public.

## Concrete next step

(1) Switch the deployment-creds override from "always overwrite"
to "only set if missing" — guard each `litellm_params[key] =
deployment_creds[key]` with a `not litellm_params.get(key)`
check. (2) Make the `model_group_name` fallback log a warning
(or remove it; the encoded ID is supposed to be a deployment ID,
not a model group). (3) Add a test that asserts caller-supplied
`api_base`/`api_key` are *not* overridden when the user passed
them explicitly. (4) Audit `data["model_id"]` flow for spend-log
exposure and either redact or document.

## Verdict

Correct fix for the routing bug, but the credential-override
policy is too aggressive and will surprise BYOK proxy operators.
Worth tightening before merge.

## What I learned

"Decode managed ID → resolve deployment → override credentials"
is the right pattern, but the *override-vs-fill* distinction is
where most multi-tenant proxy bugs live. Defaulting to overwrite
optimizes for the common case (proxy-managed creds) and silently
breaks the operator-supplied case. Defaulting to fill is the
inverse trade. Either is fine *if explicit*; what's not fine is
choosing implicitly. Same shape as the W17 drip-5 User-Agent
header collision — header set unconditionally → user override
silently dropped.
