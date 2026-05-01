# BerriAI/litellm#26963 — chore(proxy): align resource model auth checks

- **Author:** stuxf
- **Head SHA:** `f51dd68ff0607e1a95c1ceef876a7176add31536`
- **Base:** `litellm_internal_staging`
- **Size:** +814 / -98 across 9 files
- **Files changed:** `litellm/proxy/_lazy_openapi_snapshot.json` (operation_id determinism fix, dominates the diff), plus auth-check code/tests in `litellm/proxy/`

## Summary

Two-pronged proxy hygiene PR. The dominant churn in `_lazy_openapi_snapshot.json` is a bulk fix of FastAPI's non-deterministic `operationId` generation: routes that previously serialized with the wrong-method `operationId` (e.g. all six methods of `/anthropic/{endpoint}` collapsed to `..._put`, causing `operationId` collisions across the OpenAPI document) are now stamped with their actual method (`..._delete`, `..._get`, `..._patch`, `..._post`). Several `_2`-suffixed operation IDs (`test_connection_mcp_rest_test_connection_post_2`, `policies_usage_overview_..._get_2`, etc.) appear where FastAPI's de-duplication bumped them — also visible. The non-snapshot files presumably carry the actual auth-check alignment work named in the title.

## Specific code references

- `_lazy_openapi_snapshot.json:3575/3619/3663/3707`: per-method correction on `/anthropic/{endpoint}` — `delete`, `get`, `patch`, `post` operations all had `operationId: anthropic_proxy_route_anthropic__endpoint__put` (the literal name of the most recently registered handler), now correctly stamped with their own method suffix. This was a real OpenAPI-spec correctness bug — clients generating SDKs from the snapshot would have collided four operations into one.
- `_lazy_openapi_snapshot.json:13263/13302/13341/13380`: same pattern on `/langfuse/{endpoint}` — four methods all mis-stamped as `..._put`. Symmetric fix.
- `_lazy_openapi_snapshot.json:14011/14056/14101/14126`: `_2`-suffixed operation IDs on the `mcp-rest/test/connection`, `mcp-rest/test/tools/list`, `mcp-rest/tools/call`, `mcp-rest/tools/list` routes. The `_2` suffix is FastAPI's auto-de-dup behavior — these routes are registered twice (probably once on the public router and once on an internal/admin router), and the snapshot now correctly distinguishes the second registration. Worth verifying the duplicate registration is intentional rather than a router-mounting bug.
- `_lazy_openapi_snapshot.json:21899`: `policies_usage_overview_..._get_2` and `:22524 estimate_attachment_impact_..._post_2` — same auto-dedup pattern on the policies router.
- `_lazy_openapi_snapshot.json:26886-27081`: `/toolset/{toolset_name}/mcp` six-method correction — `delete`, `get`, `head`, `options`, `patch`, `post` all previously stamped as `..._put`, now per-method. Same bug class as the `/anthropic` and `/langfuse` cases. The route handler description block (`Namespace a toolset as its own MCP endpoint. ... non-admin API keys must have the toolset listed in their object_permission.mcp_toolsets grant list, or the request will be rejected with a 403`) is preserved verbatim across all six method variants — good.
- `_lazy_openapi_snapshot.json:28332`: `vector_store_list_v1_vector_stores_get_2` — another auto-dedup, same router-double-mount question as the `mcp-rest` cases.

## Reasoning

The snapshot diff is mechanical and large but the underlying fix is real correctness — non-deterministic `operationId` generation in OpenAPI dumps causes downstream SDK collisions (operations with duplicate IDs are typically dropped or merged by code generators). Pattern is the same as the operation-id determinism fix bundled into #26956 and #26966 — this is presumably the canonical landing of that fix, with the other PRs picking it up via rebase noise.

Concerns for the maintainer:
- **Router-double-registration audit.** The `_2`-suffixed operation IDs (`mcp-rest/*`, `policies_usage_overview`, `vector_store_list`, `estimate_attachment_impact`, `resolve_policies_for_context`) reveal that several routes are registered twice in the FastAPI app. This may be intentional (public + admin variants) or a bug (router mounted under two prefixes accidentally). Either way it's worth a one-line note in the PR confirming which.
- **PR title says "align resource model auth checks"** but the diff window I reviewed is dominated by the OpenAPI snapshot fix. The actual auth-check changes are presumably in other files in this 9-file PR — review of those files (out of my diff window) is the gating concern for the auth correctness claim. The snapshot churn is fine; the auth-check delta is what determines the verdict on the title's stated intent.
- **Snapshot determinism mechanism.** The fix logic (presumably in `_lazy_openapi_snapshot.py` or equivalent, similar to the `_set_deterministic_operation_ids(routes)` helper seen in #26956 / #26966) should ship as a single canonical implementation rather than three variants — verify the same helper is in use.

Verdict reflects the snapshot-only window I had: the visible delta is correct and the bug class is real, but the auth-check alignment named in the title needs out-of-band review before merge.

## Verdict

**needs-discussion**
