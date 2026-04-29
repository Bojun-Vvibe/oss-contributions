# BerriAI/litellm#26753 — Fix: drop client_metadata from provider requests

- **Repo**: BerriAI/litellm
- **PR**: [#26753](https://github.com/BerriAI/litellm/pull/26753)
- **Head SHA**: `77b6e2b0d1386f7cb8d2c1d4a43a4288cafffdd9`
- **Author**: urainshah
- **Diff stats**: +10 / −0 (2 files)

## What it does

Responses-API clients (e.g. an OpenAI-compatible CLI agent) attach a
`client_metadata` field in the request body. That field was leaking
through `**kwargs` into provider-specific request shapes — most
visibly into Bedrock's `additionalModelRequestFields`, which Bedrock
then 400s with `client_metadata: Extra inputs are not permitted`.

Fix: add `"client_metadata"` to the `internal_params` set in
`filter_internal_params` so the field is stripped before it reaches
provider transformations. Centralized fix per Greptile's review on the
prior issue (#26172) which had been attempting per-provider point fixes.

## Code observations

- `litellm/litellm_core_utils/core_helpers.py:434` — single-line
  addition of `"client_metadata"` to the `internal_params` literal set
  alongside the existing MCP-handler keys
  (`"skip_mcp_handler"`, `"mcp_handler_context"`,
  `"_skip_mcp_handler"`). The placement is correct (it IS a client-side
  passthrough that has no place reaching providers) but the set is now
  semantically heterogeneous: most members are litellm-internal
  plumbing keys, `client_metadata` is a *client*-supplied field that
  litellm intentionally drops. Worth either splitting into two sets
  (`_INTERNAL_LITELLM_KEYS` vs `_CLIENT_PASSTHROUGH_DROP_KEYS`) or
  adding a comment grouping them.
- `tests/test_litellm/litellm_core_utils/test_core_helpers.py:204-211`
  — new `TestFilterInternalParams::test_should_strip_client_metadata`
  asserts `client_metadata` is removed AND that an unrelated
  `temperature` field passes through. Two-assertion test is the right
  minimum; a parametric test covering all five members of the
  `internal_params` set would future-proof against accidental
  removals from the set.
- The PR body explicitly notes this addresses the same root cause as
  prior per-provider PR #26172 with a centralized approach. That's
  the right architectural call — Bedrock is the *visible* failure
  surface, but the same leak almost certainly affects any
  strict-schema provider (Anthropic, Vertex with strict JSON,
  OpenAI's own strict-mode tools) the moment the client metadata
  format diverges from the provider's allow-list.

## What's missing

- No test that demonstrates the original Bedrock 400 going away. A
  small fixture posting a Responses-API-shaped body through to a
  mocked Bedrock transformer and asserting the outgoing
  `additionalModelRequestFields` is empty (or at least lacks
  `client_metadata`) would be the regression-fence test.
- The `filter_internal_params` call site coverage isn't shown in the
  diff. Confirm that **every** provider request path goes through
  `filter_internal_params` before constructing the provider-specific
  body. If even one provider builds its body directly from raw kwargs,
  this fix won't help that provider and the leak silently persists.
- No CHANGELOG entry. `client_metadata` is a Responses-API-aware
  field; downstream observability/cost-attribution tools that were
  reading it off the *outbound* provider request (rather than the
  inbound litellm request) will silently lose it. Should be called
  out.

## Verdict: `merge-after-nits`

Right approach, minimal diff, addresses a real and reproducible
provider 400. Nits:

1. Convert the `internal_params` literal at `core_helpers.py:421-435`
   into a module-level `INTERNAL_PARAMS = frozenset({...})` with two
   adjacent comment-grouped sub-blocks: "litellm-internal plumbing
   keys" and "client passthrough fields to drop". This makes the
   intent of `client_metadata`'s membership explicit and lets the
   set be re-used / introspected from elsewhere.
2. Add a parametric test
   (`@pytest.mark.parametrize("key", list(INTERNAL_PARAMS))`)
   asserting every member is stripped. Catches accidental removals
   in one place.
3. Add a regression-fence test at the Bedrock transformation layer
   that pipes a Responses-API body with `client_metadata` set and
   asserts the outgoing `additionalModelRequestFields` does not
   contain it — that's the *failure mode* that motivated the PR
   and it deserves an end-to-end test, not just a unit test on the
   helper.
4. Audit grep-callsites of `**kwargs` in provider transformation
   files to verify `filter_internal_params` is called upstream of
   every body construction. If any provider builds bodies directly
   from raw kwargs the leak silently persists.
5. Add a one-line CHANGELOG/release-note: outbound provider requests
   no longer carry `client_metadata`. Downstream observers that
   were reading it off the wire will need to read it off the
   inbound litellm request instead.

## Follow-ups

- File a meta-issue: enumerate all client-side passthrough fields
  that should be litellm-only (`metadata`, `tags`, `user`,
  `client_metadata`, etc.) and document the canonical list of what's
  forwarded vs. dropped vs. transformed per provider.
- Consider a strict-mode env var (`LITELLM_STRICT_PROVIDER_BODY=1`)
  that 400s at the litellm boundary on any unknown key headed for a
  provider, instead of silently dropping. That would have caught
  this class of bug at integration time rather than user-report time.
