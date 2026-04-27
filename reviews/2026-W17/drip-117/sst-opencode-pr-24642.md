# PR #24642 — fix: ensure toolStreaming defaults off for non-Anthropic models on Anthropic SDK

- **Repo**: sst/opencode
- **PR**: #24642
- **Head SHA**: `90c50fc2367412a5aa13f0832f83e3a39e74fa63`
- **Author**: rekram1-node (Aiden Cline)
- **Size**: +4 / -1 across 1 file (closes #24627)
- **Verdict**: **merge-after-nits**

## Summary

Extends the existing `toolStreaming = false` carve-out for
`@ai-sdk/google-vertex/anthropic` to also cover `@ai-sdk/anthropic`
when the model id does not contain `"claude"` — i.e. third-party
or proxied non-Claude models routed through the Anthropic SDK
shape (corp gateways, OpenAI-compat providers fronted by an
Anthropic-style endpoint, etc.). Those models don't implement
streamed tool-use blocks the way real Claude does and the
streaming attempt produces malformed deltas / mid-stream errors.

## Specific changes

- `packages/opencode/src/provider/transform.ts:847-856` — the
  guard becomes `if (vertex/anthropic OR (npm === @ai-sdk/anthropic
  AND !id.includes("claude")))` and sets `result["toolStreaming"]
  = false`. The intent reads correctly: if the Anthropic SDK is
  driving a non-Claude model id, we cannot trust the model to
  emit Anthropic's streaming tool-use protocol, so disable it.

## Risks

1. **`includes("claude")` is a soft probe.** A future model id
   like `claude-haiku-via-proxy` would match (good); but a
   custom-named real Claude variant such as `c-3-5-sonnet-corp`
   would not (bad — would unnecessarily disable streaming). Worth
   commenting why the substring check is OK: the failure mode is
   "streaming off when it could've been on" which is degraded but
   safe, vs. "streaming on when the model doesn't support it"
   which crashes. A normalized id-prefix list (`claude-`,
   `anthropic.`, `bedrock.anthropic.`) would be more precise, but
   a substring check is acceptable as a stopgap.
2. **No regression test.** The closes-#24627 reproduction is
   "model-id stays non-claude → expect `toolStreaming: false`",
   which is one assertion against a synthesized
   `Provider.options(...)` input. The cost-to-add is trivial and
   pins both the bug and the future-Claude-rename scenario above.
3. **No `id` casing guard.** `model.api.id` is whatever the
   provider config supplies — `"Claude-3-5-Sonnet"` would slip
   past `includes("claude")`. `.toLowerCase().includes("claude")`
   would be more defensive at zero cost.

## Verdict

`merge-after-nits` — the conditional is the right shape and the
fix is one line of behavior change, but the substring check
should be lowercased and a one-line unit test should pin the
non-Claude-on-Anthropic-SDK case so the next refactor doesn't
silently regress #24627.

## What I learned

When an SDK shape is reused across providers ("Anthropic-flavored
endpoints serving non-Claude models"), feature toggles end up
keyed off *combinations* — SDK + model-id heuristics — rather
than a single dimension. The temptation is to add the carve-out
as an inline boolean expression; the durable shape is a tiny
predicate function (`isClaudeModelOnAnthropicSdk(model)`) so the
next "and also..." case has somewhere to grow.
