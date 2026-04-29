# BerriAI/litellm PR #26753 — Fix: drop client_metadata from provider requests

- Repo: BerriAI/litellm
- PR: https://github.com/BerriAI/litellm/pull/26753
- Head SHA: `77b6e2b0d1386f7cb8d2c1d4a43a4288cafffdd9`
- Author: urainshah (Urain Ahmad Shah)
- Size: +10 / −0, 2 files

## Context

Responses-API clients (the PR author calls out a `[redacted]/codex-cli`
pattern of `client_metadata: {...}` being attached to outbound requests)
attach a `client_metadata` field on the request body. LiteLLM treats
`**kwargs` as forward-everything by default, so that field leaks into the
provider call — and Bedrock in particular folds unrecognized kwargs into
`additionalModelRequestFields`, which Bedrock then rejects with
`400: client_metadata: Extra inputs are not permitted`. This is the
generic "provider strict mode rejects an unknown LiteLLM-internal field"
class of bug. The PR's framing as "centralized approach (per Greptile
review feedback)" — versus an alternative PR #26172 — is the right call:
fix once at `core_helpers.filter_internal_params` instead of patching every
provider's transformer.

## What the diff does

Single line of production code at
`litellm/litellm_core_utils/core_helpers.py:431-435`: adds
`"client_metadata"` to the existing `_internal_params` set used by
`filter_internal_params`. That set already houses other LiteLLM-internal
keys like `skip_mcp_handler`, `mcp_handler_context`, `_skip_mcp_handler`,
so the addition is contractually similar.

Test at `tests/test_litellm/litellm_core_utils/test_core_helpers.py:206-212`
imports `filter_internal_params` and pins:

```python
class TestFilterInternalParams:
    def test_should_strip_client_metadata(self):
        data = {"temperature": 0.7, "client_metadata": {"app": "[redacted]-cli"}}
        result = filter_internal_params(data)
        assert "client_metadata" not in result
        assert result["temperature"] == 0.7
```

That asserts both stripping and non-collateral-damage on a sibling key.

## What I'd push back on

1. **`client_metadata` is not actually a LiteLLM-internal field.** The
   other entries in `_internal_params` (`skip_mcp_handler` etc.) are
   keys that LiteLLM itself attaches inside the request pipeline. By
   contrast, `client_metadata` is something the *client* attaches —
   specifically, the OpenAI Responses API has a documented
   `metadata` / `client_metadata` field that *is* meant to be forwarded
   to OpenAI for observability and customer tagging. Dropping it
   silently in `filter_internal_params` will break that flow for any
   user who legitimately wants OpenAI to see their `client_metadata`
   (e.g. cost-allocation tagging on the OpenAI dashboard). The right
   fix is more nuanced: strip `client_metadata` only when forwarding
   to providers that *don't* accept it (Bedrock, Vertex, etc.), and
   pass it through to OpenAI / Azure-OpenAI / providers whose schema
   includes it. A per-provider allow/deny list is the correct shape,
   not a global strip.

2. **No test for the "passes through to OpenAI" case.** Even if the
   intent really is to drop globally, a test that demonstrates the
   key is *not* forwarded to OpenAI either is missing. Adding it would
   surface the regression I'm worried about above.

3. **The added test uses a literal `"[redacted]-cli"` value but doesn't
   exercise the actual leak path.** The test pins
   `filter_internal_params` in isolation, which is fine for the unit, but
   the original bug was a *call-site* bug — `filter_internal_params` was
   either not being called on the request payload before the Bedrock
   transform, or was being called but not on the right object. A higher-
   level test that fires a request with `client_metadata` against a mocked
   Bedrock client and asserts the field never reaches
   `additionalModelRequestFields` would actually pin the fix in the place
   the bug lived.

4. **Hardcoded set membership doesn't scale.** Each new "this provider
   rejects unknown fields" bug will result in another single-line addition
   to `_internal_params`. A small structured registry —
   `_PROVIDER_REJECT_FIELDS: dict[str, set[str]]` plus
   `_LITELLM_INTERNAL_FIELDS: set[str]` — would make the difference
   between "internal" and "provider-incompatible" explicit and avoid
   over-stripping for providers that accept the field.

5. **PR body's "Relevant issues / Addresses the same issue as #26172"
   needs a CHANGELOG entry calling out the silent-drop, since users with
   working `client_metadata` flows on OpenAI may now see their tagging
   disappear.**

## Suggestions

- **Reconsider the global strip.** Make this a per-provider allow-list /
  deny-list (most importantly: keep forwarding to OpenAI Responses).
- Add a unit test that pins `client_metadata` is forwarded *to* OpenAI
  but stripped *from* Bedrock.
- Add an integration-shaped test at the actual Bedrock transform call
  site so the fix can't silently regress if the call to
  `filter_internal_params` is ever moved.
- CHANGELOG entry under "Removed/Changed" so existing OpenAI users
  expecting `client_metadata` forwarding are warned.

## Verdict

**needs-discussion** — the immediate Bedrock 400 goes away, but treating a
provider-level forwarding field as a LiteLLM-internal one will silently
break OpenAI users who depend on `client_metadata` for customer tagging.
The right shape is a per-provider field policy, not a global strip. Worth
a maintainer call before merge.
