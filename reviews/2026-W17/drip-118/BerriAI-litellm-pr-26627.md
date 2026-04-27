# PR #26627 — fix(bedrock): preserve cache_control for ARN-based models in /v1/messages adapter

- **Repo**: BerriAI/litellm
- **PR**: #26627
- **Head SHA**: `2d5163e6`
- **Author**: harryzsh (HarryZhu)
- **Size**: +3 / -1 across 1 file
- **Verdict**: **merge-after-nits**

## Summary

One-line predicate widening in `is_anthropic_claude_model()` so
that Bedrock Application Inference Profiles (ARN-shaped model
identifiers like `arn:aws:bedrock:us-east-1:ACCOUNT:application-inference-profile/ID`)
get correctly classified as Claude models when routed through
the Anthropic `/v1/messages` adapter. Without this, the existing
substring check (`"anthropic" in model_lower or "claude" in
model_lower`) returns `False` for an ARN — the string contains
neither token — so the adapter silently strips `cache_control`
directives, prompt caching never activates, and users on
Application Inference Profiles pay the full uncached rate plus
the higher uncached latency. The PR includes CloudWatch numbers
that make the cost impact concrete: `cacheWriteInputTokens` goes
from 0 → 34,004 on the same workload after the patch.

## Specific changes

- `litellm/llms/anthropic/experimental_pass_through/adapters/transformation.py:719-722` — extends the boolean OR chain with `or ("arn:" in model_lower and "bedrock" in model_lower)`. Both substrings required (not just `"arn:"`) so that AWS ARNs for non-Bedrock services (e.g. `arn:aws:lambda:...`, which would never reach this code path in practice but defends against future routing changes) still classify as not-Claude.
- The same file's docstring at `:712-716` mentions `vertex_ai/*claude*` already routes through the existing `"claude" in model_lower` branch — the PR doesn't update that docstring to mention the new ARN branch, so the next reader has to discover the third clause from the diff alone.

## Risks

- **The ARN check trusts that any `arn:...bedrock...` is a Claude model**: this is true today for the `/v1/messages` Anthropic endpoint because the routing into `LiteLLMMessagesToCompletionTransformationHandler` is exclusively Anthropic-shape requests. But Bedrock hosts non-Claude models too (Amazon Titan, Meta Llama, Cohere Command, AI21 Jurassic, etc.), and if a future routing change ever sends a non-Claude Bedrock ARN through this adapter, the predicate now misclassifies it as Claude. The PR description's safety argument ("the only code path reaching this function with a Bedrock ARN is through the `/v1/messages` Anthropic endpoint") is correct today, but the predicate doesn't enforce it — it relies on a routing invariant living in callers. A safer shape would be to inspect the ARN's `application-inference-profile` slug or to keep a registry of Application Inference Profile → underlying model mappings, but that's a larger change than the bug warrants.
- **No new test in the PR**: the existing test suite presumably covers `"claude" in model` and `"anthropic" in model` paths, but a one-line addition of `("arn:aws:bedrock:us-east-1:1234567890:application-inference-profile/abc123", True)` to whatever parameterized fixture covers `is_anthropic_claude_model` would pin the new behavior at zero cost. As-is, the next refactor that consolidates this predicate could silently drop the ARN clause and the only signal would be a regression report from a Bedrock-on-`/v1/messages` user.
- **Verification was done via Bedrock CloudWatch on a live deployment**: the cache token jump from 0 → 34,004 is convincing as a real-world signal but doesn't prove the code path under unit test. The author should add at least the `is_anthropic_claude_model("arn:aws:bedrock:...")` direct assertion plus an end-to-end test that round-trips a `cache_control` directive through `LiteLLMMessagesToCompletionTransformationHandler` for an ARN model and asserts the resulting `cachePoint` block survives.
- **Case sensitivity**: the function does `model.lower()` first, so `ARN:AWS:BEDROCK:...` works. AWS ARNs are technically lower-case by convention but that's the only line of defense against a stray uppercase ID; worth keeping the `.lower()` even after a future refactor.

## Verdict

`merge-after-nits` — the bug is real, the cost impact is concrete
(no cache discount on a workload that should be heavily cached),
and the fix is the minimal change that closes it. Two asks before
merge: (1) add a unit test entry for `is_anthropic_claude_model`
covering both an Application Inference Profile ARN and a
hypothetical non-Claude Bedrock ARN to pin the predicate's
intent; (2) update the function docstring at `:712-716` to
mention the ARN+Bedrock branch so the next reader doesn't have
to learn it from `git blame`. The maintainer might also consider
filing a follow-up to tighten the predicate from "any
Bedrock ARN" to "Bedrock ARNs known to host Claude" once a
registry exists.

## What I learned

This is a clean example of an opaque identifier breaking a
substring-based capability check, and the `/v1/messages`
adapter is an interesting pinch point because it sits between
the Anthropic wire format (which has `cache_control`) and the
internal OpenAI-like representation (which has its own caching
shape). The "is this a Claude model?" predicate is doing
double duty: it both gates the cache_control translation and
implicitly assumes the routing layer wouldn't send a non-Claude
model through this adapter. Splitting that into "should we
preserve cache_control?" (a transform-level decision) and "is
this routing valid?" (a router-level decision) would let each
layer be checked independently. For now the substring widening
is the right small-scope fix.
