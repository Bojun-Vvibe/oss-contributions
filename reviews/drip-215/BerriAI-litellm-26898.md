# Review: BerriAI/litellm#26898 — fix: allow S3 callback health test

- PR: https://github.com/BerriAI/litellm/pull/26898
- Head SHA: `9f1f7f4b95d84f8346d27700f91a91be7ff3ecb9`
- State: OPEN
- Files: `litellm/proxy/health_endpoints/_health_endpoints.py` (+2/-0), `tests/test_litellm/proxy/health_endpoints/test_health_services_callbacks.py` (+17/-0, new file)
- Verdict: **merge-as-is**

## What it does

Adds `"s3"` to two parallel allow-lists in the `health_services_endpoint`
handler — the type-annotation `Literal[...]` at `_health_endpoints.py:127` and
the runtime guard `if service not in [...]` at `:202` — so that a request to
`/health/services?service=s3` no longer 400s with "service not in allowed
list" when the proxy operator has configured S3 as a success callback.

## Notable changes

- `_health_endpoints.py:127` — `"s3"` inserted between `"arize"` and
  `"sqs"` in the `Literal[...]` annotation that types the `service` query
  parameter. Alphabetically out of order (s comes after sqs and before
  the rest is mixed), but the existing list isn't sorted either —
  `["webhook", "email", ..., "datadog_llm_observability", "generic_api",
  "arize", "sqs"]` is roughly category-grouped. The insertion is between
  the cloud-observability cluster (`arize`) and the queue cluster (`sqs`),
  which is the right neighbourhood (S3 is a destination/sink like SQS).
- `_health_endpoints.py:202` — same string added to the runtime guard
  list at the same position. The two lists must stay in sync; the PR
  correctly updates both.
- `tests/.../test_health_services_callbacks.py:1-17` — new file with one
  async test that patches `litellm.success_callback` to `["s3"]`, mocks
  `litellm.acompletion` to a no-op AsyncMock, calls
  `health_services_endpoint(service="s3")`, and asserts (a) `result["status"]
  == "success"` and (b) `"s3" in result["message"]`. The `acompletion` mock
  is the right shape because the health-services handler does a probe
  completion call as part of the integration check; mocking it avoids a
  network round-trip while still exercising the dispatch path.

## Reasoning

This is the lowest-risk class of change in litellm: adding a string to a
duplicate allow-list pair so an existing integration becomes addressable via
the health endpoint. The S3 callback itself has been in litellm for a long
time (`litellm.success_callback` accepts `"s3"` and writes spend logs to
S3); the gap was that operators couldn't probe its health without hitting
the same allow-list error every other recently-added service hit before it
was whitelisted (datadog_llm_observability and arize were both added in the
same shape recently per git history).

The two-list duplication is a known wart — the type annotation and the
runtime guard should share a single constant — but fixing that is out of
scope for a one-service-name addition. As long as both sites stay in sync
the bug stays fixed; the new test exercises the runtime guard arm so a
future divergence would surface there.

The new test doesn't actually probe S3 — it mocks `acompletion` and just
verifies the dispatch path returns `success` with `s3` in the message. That
is the right scope for the PR's contribution: the actual S3 health-check
logic (presumably bucket-write probe) is tested elsewhere, and what's new
here is the gate-passing behaviour.

## Nits (non-blocking)

1. The two parallel allow-lists at `:124-132` and `:199-207` are an
   obvious duplication smell — extract a module-level
   `_HEALTH_CHECK_ALLOWED_SERVICES: tuple[str, ...]` and reference it from
   both the `Literal` annotation (`Literal[*_HEALTH_CHECK_ALLOWED_SERVICES]`
   in 3.11+) and the runtime guard. Out of scope here, but the next time
   someone adds a service they should propose this.
2. The list is not sorted. As it grows past 10 items a sort would help
   reviewers confirm a service is or isn't already in the list at a glance.
3. The new test doesn't pin the `service` not in allowed list error path —
   `await health_services_endpoint(service="not_a_real_service")` should
   raise `HTTPException`. A second 4-line negative test would protect the
   gate from being silently widened.
4. `from unittest.mock import AsyncMock, patch` imports `AsyncMock` but
   only uses it as a `new=AsyncMock(return_value={})`. The empty-dict
   return is a slight semantic mismatch with `litellm.acompletion`'s real
   return shape (`ModelResponse`); if the handler ever starts inspecting
   the response, this test will need updating. Worth a one-line comment
   pinning the assumption ("`acompletion` return is intentionally minimal —
   handler only checks for non-exception").
