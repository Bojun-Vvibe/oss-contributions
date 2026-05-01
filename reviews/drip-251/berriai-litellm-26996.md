# BerriAI/litellm#26996 — chore(security): close two unaddressed SSRF cases

- **PR:** https://github.com/BerriAI/litellm/pull/26996
- **Head SHA:** `90fd791e0d4c794bd009ca0d84c558d9ff3d604a`
- **Stats:** +610 / -69 across 43 files (4 source modules, 3 test files, 1 frontend bundle rebuild touching ~36 `out/**/index.html` files; the substantive changes are 4 source files + 3 test files)
- **Refs:** VERIA-51 (the in-comment ticket; PR body says "operational hardening" with no public issue link)

## Context
Two thematically-similar SSRF holes both letting a caller exfiltrate the operator's provider API keys to an attacker-controlled host. They're independent failure modes but get closed in one PR because both close on the same trust-boundary discipline (recursive validation of nested config, same-origin assertion on follow-up URLs).

## Design — Hole 1: nested-dict smuggling past `is_request_body_safe`
`auth/auth_utils.py` (~99 lines added, ~65 deleted) hardens `is_request_body_safe`. Background: the validator already blocks banned params like `api_base`, `api_key`, `langfuse_host`, `s3_endpoint_url` at the *root* of the request body, gated by `general_settings.allow_client_side_credentials` or per-deployment `configurable_clientside_auth_params`. But the Milvus vector-store transformer unpacks `litellm_embedding_config` into `litellm.embedding(**embedding_config)`, so a caller can stash `api_base` *inside* the nested dict and bypass the root-level check.

Fix: single-level recursion that applies the same banned-param check (and the same admin opt-in) to known-nested-config dicts — currently just `litellm_embedding_config`, with the comment that future entries get added here. Critically, the admin-side vector-store config flows through `litellm_params` (not the request body), so admin-configured Milvus deployments don't see this validator and remain unaffected. That's the right scoping — admin trust boundary is unchanged, caller trust boundary is now consistent across nesting levels.

## Design — Hole 2: cross-origin polling URLs receive operator API keys
Five sites in three providers blindly fetched the polling URL returned in the upstream's response *and attached the operator's API key*: Azure DALL-E 2 sync + async (`llms/azure/azure.py:899-925, 1009-1052`), Azure Document Intelligence sync + async (`llms/azure_ai/ocr/document_intelligence/transformation.py:600-616, 711-732`), Black Forest Labs image generation sync + async (`llms/black_forest_labs/image_generation/handler.py:317-329, 414-426`), BFL image edit sync + async (`llms/black_forest_labs/image_edit/handler.py:331-343, 416-428`).

The fix introduces `assert_same_origin(candidate_url, expected_url)` at `litellm_core_utils/url_utils.py:202-241` — scheme + case-insensitive host + port (with default-port normalization 80/443). Three correctness points worth quoting:

```python
if candidate.scheme not in _ALLOWED_SCHEMES:
    raise SSRFError("URL scheme is not allowed")
if candidate.scheme != expected.scheme:
    raise SSRFError("Origin mismatch on scheme")
candidate_host = _normalize_host(candidate.hostname or "")
expected_host = _normalize_host(expected.hostname or "")
if not candidate_host or candidate_host != expected_host:
    raise SSRFError("Origin mismatch on host")
default_port = 443 if candidate.scheme == "https" else 80
candidate_port = candidate.port if candidate.port is not None else default_port
expected_port = expected.port if expected.port is not None else default_port
if candidate_port != expected_port:
    raise SSRFError("Origin mismatch on port")
```

The docstring explicitly notes: error messages identify *which* component mismatched but never echo the operator's `expected` host or the candidate's hostname back to the caller — "in the SSRF threat model the caller is the attacker, and reflecting host info would be a secondary leak of operator infrastructure details." Right discipline.

## Design — Hole 2b: Azure DALL-E 2 "Got=..." reflection
The Azure DALL-E 2 sync + async paths used to raise `Exception("Expected 'status' in response. Got={}".format(response.json()))` on the missing-status case at `azure.py:1019-1021`. That turned Blind SSRF into Full-Read SSRF — when the polling URL points at an internal JSON API (cloud metadata service, internal admin endpoint), the response body got reflected back to the attacker through the exception message. Sanitised to `AzureOpenAIError(status_code=502, message="Polling response missing 'status' field")`. Inline comment at `:1043-1046` explicitly names the threat model.

## Tests
- `tests/test_litellm/litellm_core_utils/test_url_utils.py:+73` — 7 tests covering scheme mismatch, host mismatch, port mismatch, default-port normalization, case-insensitive host, non-HTTP scheme rejection.
- `tests/test_litellm/proxy/auth/test_auth_utils.py:+126` — 6 tests covering root-level vs nested banned-param enforcement, admin opt-in passthrough, non-dict tolerance for the recursion (so a non-dict `litellm_embedding_config` doesn't crash the validator).
- `tests/test_litellm/llms/test_polling_url_origin_match.py:+177` — 5 cross-origin rejection tests across Azure DI sync, BFL image-gen sync+async, BFL image-edit sync, plus one same-origin sanity check confirming legitimate polling still proceeds.

148/148 in the affected suites pass per PR body.

## Risks
- **No async-path coverage for Azure DALL-E** in `test_polling_url_origin_match.py` — sync DI and BFL async are covered but not Azure DALL-E async. That's the highest-volume Azure path and least likely to ship without a real-world cross-origin response, but a one-test addition would lock it.
- **No test for the sanitised "Got={}" exception** — the Full-Read-SSRF severity reduction is the highest-impact change in this PR and it has no regression test. Add one that mocks a polling response with `{"foo": "secret"}` and asserts the raised `AzureOpenAIError.message == "Polling response missing 'status' field"` (no `secret` substring).
- **Single-level recursion only** — `is_request_body_safe` recurses one level into `litellm_embedding_config`. If a future nested-config dict is introduced and forgotten, the same bug returns. Worth either a registry/decorator pattern or at minimum a unit test that enumerates known nested-config keys against a hardcoded list to catch additions.
- **Frontend rebuild touches ~36 `out/**/index.html` files** — pure build artifacts but they balloon the diff and make code review noisier than the security fix warrants. Worth a `.gitignore` entry for `litellm/proxy/_experimental/out/` if it's not strictly required to commit.
- **Allowlist option for legitimate cross-origin polling** — PR body acknowledges no current provider returns cross-origin polling URLs, but if one shows up, the fix becomes "add an allowlist". Consider stubbing the allowlist hook now (typed `cross_origin_polling_allowlist: List[str]` config field, default empty) so the future addition isn't a separate API change.

## Suggestions
1. **Add the Full-Read-SSRF regression test** for the sanitised "Got=" exception — highest-impact missing test.
2. **Add Azure DALL-E async cross-origin rejection test** for symmetry with sync.
3. **Stub the cross-origin allowlist hook** with default-empty so adding a legitimate cross-origin polling provider later doesn't require an API surface change.
4. **Registry pattern for nested-config keys** so the single-level recursion is self-extending: a `KNOWN_NESTED_CONFIG_KEYS = {"litellm_embedding_config", ...}` set referenced at the recursion site, plus a CI-level audit ensuring new top-level keys are evaluated for nesting.
5. **Frontend bundle commit** — confirm with maintainers whether `out/**/index.html` should be in the diff or `.gitignore`d.

## Verdict
**merge-after-nits** — two real security holes closed at the right boundary (recursive validation gated by the same admin opt-in, typed `assert_same_origin` with non-reflective error messages applied at all 5 polling sites, sanitised exception that closes the Blind→Full-Read SSRF escalation), with strong dispositive tests for the new helper. Two missing tests (Full-Read regression, Azure DALL-E async) and the registry-pattern question for future nested keys are the gaps worth closing before merge.

## What I learned
The "validator runs at the root, but the consumer un-nests one level" pattern is a recurring SSRF / privilege-escalation footgun — same shape as XML entity expansion attacks, JSON merge-patch bypasses, etc. The right fix is *recursive validation gated by the same trust check*, not "add the nested key to a separate banned list" (which drifts). And the "raw exception message reflects upstream response body" pattern is one of the silent escalators from Blind SSRF to Full-Read SSRF — sanitise *every* exception that includes upstream content, even when the SSRF surface itself is closed, because the next regression might re-open the SSRF and the exception message is the residual leak. The non-reflective error message contract in `assert_same_origin` ("never echo the operator's host or the candidate's hostname back") is also worth internalising — error messages are an exfiltration channel.
