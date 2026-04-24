# PR #26416 — fix(auth): enforce model ACL on Bedrock passthrough routes

**Repo:** BerriAI/litellm  
**Surface:** `litellm/proxy/auth/auth_utils.py` (regex + reverse-lookup
helper, extended `get_model_from_request`); `auth_checks.py`,
`user_api_key_auth.py` plumbing; new
`TestGetModelFromRequestBedrockPassthrough` (12 cases).  
**Severity:** High (silent ACL bypass on a passthrough surface)

## What changed

Bedrock passthrough routes
(`/bedrock[/v{N}]/model/{modelId}/(invoke|invoke-with-response-stream|converse|converse-stream|count_tokens|count-tokens)`)
carry the model exclusively in the URL — there is no top-level `model`
field in the native InvokeModel/Converse request body. The pre-PR
`get_model_from_request()` therefore returned `None`, and downstream
`can_key_call_model` / `common_checks` had nothing to enforce against.
A virtual key scoped to a small set of models could call *any* Bedrock
model via these routes.

The PR:
1. Adds a regex that captures `modelId` from the path.
2. Adds `_resolve_bedrock_model_id_to_model_group()` that walks
   `llm_router.get_model_list()`, strips `bedrock/` or `bedrock/converse/`
   prefixes from each deployment's `litellm_params.model`, and
   returns the matching deployment's `model_name` (the user-facing
   model_group). This is the value the existing ACL check operates on.
3. Threads `llm_router` through 3 call sites of `get_model_from_request`
   plus the `common_checks` call site.
4. 12 unit tests in `TestGetModelFromRequestBedrockPassthrough`.

## What's risky / wrong / missing

1. **Fail-open on missing router.** When `llm_router is None`,
   `_resolve_bedrock_model_id_to_model_group` returns `None`, and the
   caller falls back to `model = resolved or bedrock_model_id`. The
   raw Bedrock id then flows into `can_key_call_model`. If the key's
   allow-list contains user-facing names like `claude-opus-4-7` (no
   `bedrock/` prefix), the raw id `global.anthropic.claude-opus-4-7`
   will *not* match, so the request is correctly denied. But if a key
   explicitly allows raw Bedrock ids — which is a supported config —
   the bypass remains. Worth adding an integration test for the
   "router is None during early boot / hot reload" race. At minimum,
   document which path is intentional.

2. **Regex `.+` is greedy.** The capture group `(?P<model>.+)` will
   greedily consume slashes, so a path like
   `/bedrock/model/foo/bar/converse` resolves modelId = `foo/bar`. The
   author calls this out (Bedrock modelIds can contain slashes for
   inference-profile and `application-inference-profile/...` cases).
   But this means a *malformed* path like `/bedrock/model/foo/converse/converse`
   would resolve modelId = `foo/converse` — strange but harmless.
   Consider `(?P<model>[^?]+?)` (non-greedy, anchored) and let the
   alternation handle the trailing action token.

3. **`get_model_list()` may be expensive.** Each Bedrock passthrough
   request now does a linear scan over all router deployments inside
   the auth path. For deployments with hundreds of models this is per-
   request CPU work on the hot auth path. Cache the reverse map in the
   router (or memoize on a model-list version counter).

4. **Exception swallowing is too broad.**
   ```python
   try:
       deployments = llm_router.get_model_list() or []
   except Exception:
       return None
   ```
   A `get_model_list()` that fails for any reason silently fails *open*
   (returns `None`, raw modelId flows through, ACL may pass). A narrow
   `except (AttributeError, RouterError)` plus a `verbose_logger.error`
   would convert this from "bypass on bug" to "denied with diagnostic."

5. **Coverage gap on inference profiles.** The PR description mentions
   `application-inference-profile/...` prefixes. Need an explicit test
   that a key allowed for the *underlying* model group can still hit
   the inference-profile id path; otherwise this fix breaks a documented
   Bedrock pattern.

6. **The 4 plumbing call sites are not symmetric.** Three sites in
   `user_api_key_auth.py` pass `llm_router=llm_router`, plus
   `common_checks` in `auth_checks.py`. Audit whether any *other*
   call to `get_model_from_request` exists (e.g. logging, spend
   tracking, fallback resolution) — those sites will continue to see
   `None` for Bedrock passthrough and may silently mis-attribute spend
   or skip cost-zero shortcuts.

## Suggested fix

Beyond what's in the PR:
- Memoize the reverse-lookup map keyed on a router-version sentinel.
- Tighten the exception class in `_resolve_bedrock_model_id_to_model_group`
  and log on failure.
- Add tests for: (a) router-None path, (b) inference-profile path,
  (c) spend-attribution / logging sites if those also call
  `get_model_from_request`.
- Confirm the regex anchoring is correct against malformed/adversarial
  paths.

## Severity

High. ACL bypass on a public passthrough surface, with the caveat
that exploitation requires the deployment to actually have Bedrock
models routable. Fix is well-scoped and the test bench is the most
thorough I've seen on a litellm auth fix this week. Land it; the
defects above are follow-ups, not blockers.
