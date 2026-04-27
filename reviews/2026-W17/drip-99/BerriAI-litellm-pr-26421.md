# BerriAI/litellm#26421 — fix(proxy): apply user.models restriction to GET /v1/models list

- **Head**: `d156f3320965c926647a442f827c184d6f76f3e0`
- **Size**: +734/-50 across 14 files
- **Verdict**: `request-changes`

## Context

Fixes #26420: a `user.models` access-restriction leak on the model-list
endpoint. The proxy correctly enforces `user.models` on
`/chat/completions` (call-time), but `GET /v1/models` (discovery) ignores
it — so a user restricted to the `common-models` access group sees the
full proxy model list, even though calling a non-allowed model returns
401. The user-visible bug is "I see model X in the dropdown, picking
it gives 401" which is confusing and a minor information leak about the
existence of internal models.

## Design analysis

### The actual bug fix is small and correct

`litellm/proxy/auth/model_checks.py:207+` adds a new helper:

```python
def get_user_models(
    user_models: List[str],
    ...
) -> List[str]:
    """
    Returns expanded model list from user.models after resolving access groups.
    """
    if not user_models:
        return []
    all_models: List[str] = list(user_models)
    ...
```

And the caller in `litellm/proxy/utils.py:get_available_models_for_user`
now consults `user.models` after the existing key/team/proxy chain:

```python
from litellm.proxy.auth.model_checks import get_user_models
expanded_user_models = get_user_models(
    user_models=user_object.models,
    ...
)
all_models = [m for m in all_models if m in set(expanded_user_models)]
```

Plus the access-group expansion contract pinned by tests at lines 622+:

- `test_get_user_models_empty_means_unrestricted` — empty list returns
  empty (caller treats this as "no filter").
- `test_get_user_models_expands_access_group` — `["common-models"]`
  expands to the actual model list from that group.
- `test_get_user_models_no_default_models_returns_sentinel` — handles
  the "no default models" sentinel marker correctly.
- `test_get_user_models_all_proxy_models` — `all-proxy-models` is the
  unrestricted-passthrough sentinel.
- `test_get_user_models_deduplicates` — overlapping access groups
  don't double up models.

The five-test matrix is the right shape for an access-group resolver:
empty / explicit-list / sentinels-(no-default, all-proxy) / dedup. The
intersection logic (`[m for m in all_models if m in set(expanded_user_models)]`)
correctly preserves the existing key/team/proxy chain *and* further
narrows by `user.models`, which matches the precedence the bug ticket
asks for.

If this were the only change in the PR, this would be a clean
`merge-after-nits` — the bug fix is well-scoped, well-tested, and
correctly pinned at the contract level.

## The blocking concern: PR scope is much larger than the title

The PR title says "fix(proxy): apply user.models restriction to GET
/v1/models list" but the diff touches 14 files and ~784 lines, including
material changes to subsystems that have nothing to do with the
`/v1/models` access-control bug:

### 1. `ModifyResponseException` relocation (`litellm/exceptions.py`, `integrations/custom_guardrail.py`)

The exception class is moved from `custom_guardrail.py` to
`exceptions.py`, with `custom_guardrail.py` re-exporting it via
`from litellm.exceptions import ModifyResponseException as ModifyResponseException`.
This is a guardrail-subsystem refactor. It's defensible (canonical
exceptions belong in `exceptions.py`) but it changes the import surface
and has nothing to do with #26420.

### 2. Bedrock cache-control TTL extension for Claude 4.5
(`prompt_templates/factory.py:5061-5230`,
`bedrock/chat/converse_transformation.py:1302/1370`,
`bedrock/messages/invoke_transformations/anthropic_claude3_transformation.py:135-167`)

```python
def add_cache_point_tool_block(
    tool: dict, model: Optional[str] = None
) -> Optional[BedrockToolBlock]:
    from litellm.llms.bedrock.common_utils import is_claude_4_5_on_bedrock
    ...
    if (ttl in ["5m", "1h"]
        and model is not None
        and is_claude_4_5_on_bedrock(model)):
        cache_point_block["ttl"] = ttl
    return {"cachePoint": cache_point_block}
```

This is a substantive feature: support for non-default cache TTLs
(`5m`/`1h`) on Claude 4.5 via Bedrock. It threads `model` through
`_bedrock_tools_pt(...)` and `_process_tools_and_beta(...)`. It also
extends `_remove_ttl_from_cache_control` to process the `tools`
content list (previously only `system` and `messages`).

This is a reasonable feature, with its own test file
(`test_anthropic_claude3_transformation.py`) and prompt-template tests,
but it is **completely orthogonal** to the user.models bug.

### 3. Unified guardrail / passthrough endpoint changes
(`unified_guardrail/unified_guardrail.py`,
`pass_through_endpoints/pass_through_endpoints.py`,
`tests/test_litellm/proxy/pass_through_endpoints/test_passthrough_post_call_guardrails.py`)

Touches the unified-guardrail integration and the pass-through endpoint
post-call guardrails path. Without seeing the full hunks I can't tell
whether these are bug fixes or features, but they're a third
independent thread.

### 4. Auth-time `key.models` enforcement on Bedrock passthrough

The body's title — "apply user.models restriction" — and the diff's
`/v1/models` filtering are about the *list* endpoint, but other tests
in the diff (`test_model_management_endpoints.py`,
`test_model_checks.py`) appear to also touch *call-time* enforcement.
That's a fourth potential thread.

## Why this is `request-changes`

Bundling four orthogonal changes in one PR has concrete review costs:

1. **Bisect difficulty.** If any of these subsystems regresses six weeks
   from now, the bisected commit is a 14-file diff in which only one
   file is responsible. The user.models fix is the title; the actual
   regression could be in the cache-TTL plumbing or the
   `ModifyResponseException` relocation.

2. **Test scope.** Each of the four threads has its own tests, which is
   good — but the reviewer has to mentally partition "which test pins
   which thread" and verify each subsystem's contract in isolation.
   That's the kind of cognitive load that makes reviewers either
   rubber-stamp or reject wholesale.

3. **Cherry-pick / backport granularity.** If a downstream consumer
   wants the user.models fix but is on a release branch that doesn't
   have Bedrock Claude 4.5 support yet, they can't cleanly cherry-pick
   this PR.

4. **Title/description mismatch.** The PR title and `Fixes #26420`
   issue link both narrowly describe the user.models fix. A future
   reader doing `git log --grep="user.models"` will find this commit
   and assume that's the only thing it does — and miss that the same
   commit also extended Bedrock cache-control TTL.

The right shape is to split into ~3–4 PRs:

- **PR A (title matches scope)**: just the
  `model_checks.py` + `proxy/utils.py` + corresponding tests for the
  `user.models` filter on `/v1/models`. This is what #26420 asks for.
- **PR B**: `ModifyResponseException` relocation. Pure refactor;
  trivially mergeable on its own.
- **PR C**: Bedrock Claude 4.5 cache-control TTL feature with its
  own issue link.
- **PR D**: Unified guardrail / pass-through endpoint changes (if
  they're a coherent fix).

Each lands faster, each is easier to revert, each is easier to
backport.

## Concerns / nits (on the user.models fix itself, scoped narrowly)

Assuming the user.models fix is split into its own PR:

1. **Set construction in the filter loop**: `[m for m in all_models if m in set(expanded_user_models)]`
   — the `set()` is constructed once per call (Python comprehensions
   evaluate the `if` clause's RHS once), so this is fine, not the
   per-iteration set rebuild it superficially looks like. Could still
   hoist to `expanded = set(expanded_user_models)` on the prior line
   for readability.

2. **`get_user_models` signature** — `user_models: List[str]` is fine
   but the docstring should explicitly say "an empty list means
   unrestricted (caller skips the filter)" because the test name
   `empty_means_unrestricted` is the only place that contract is
   pinned, and a future change to "empty list = no models allowed"
   would be a silent breaking change.

3. **No test for the interaction of `user.models` and `key.models`**
   restrictions when both are set. The body says key.models is
   already filtered upstream, so the chain is
   key→team→proxy→user_intersect; a test pinning `user.models = ["A","B"]`
   AND `key.models = ["B","C"]` → result `["B"]` would lock down the
   intersection contract.

## Verdict reasoning

The user.models fix itself is correct, well-tested, and well-scoped at
the file level — that part of the PR alone would be `merge-after-nits`.
But the PR bundles three other orthogonal changes (Bedrock cache-control
TTL feature, guardrail exception relocation, unified guardrail /
pass-through changes), each substantial enough to warrant its own
review and CI-bisect target. Recommendation: split. Once the user.models
fix is in its own PR, fast-track it; the cache-TTL feature deserves
its own review focused on Bedrock-specific concerns.

## What I learned

"PR title narrowly describes one fix; PR diff bundles three more
unrelated changes" is a recurring repo-hygiene pattern in actively
maintained provider-aggregator codebases like LiteLLM, where the
maintainer is fixing things across the codebase faster than they can
publish individual PRs. The split-PR ask is unpopular because it
costs the author time, but the cost is paid back many times over in
bisect-ability and backport-ability. The narrow user.models fix here
is exactly the kind of small, focused contract change that *should* be
trivially mergeable on its own — and that's the case I'd make to the
author for the split.
