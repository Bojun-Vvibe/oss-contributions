# charmbracelet/crush#2568 — feat: support Bedrock cross-region inference model IDs and annotate 1M models

- PR ref: `charmbracelet/crush#2568` (https://github.com/charmbracelet/crush/pull/2568)
- Head SHA: `e44f712255b45276e46d108879860258a96d61bf`
- Title: feat: support Bedrock cross-region inference model IDs and annotate 1M models
- Verdict: **merge-after-nits**

## Review

Two small, independently useful changes bundled together cleanly. The
cross-region-inference fix in `internal/config/load.go:115-120` is exactly right:
AWS publishes model IDs like `us.anthropic.claude-opus-4-6-v1:1m` and
`eu.anthropic.claude-sonnet-4-6:1m` for cross-region inference profiles, and the
old `strings.HasPrefix(model.ID, "anthropic.")` check at the previous line 280
rejected all of them with a misleading "only supports anthropic models" error.
The new helper `isBedrockAnthropicModel` accepts both shapes via
`HasPrefix("anthropic.") || Contains(".anthropic.")`, which is tight enough that
unrelated IDs like `openai.gpt-4` still fail (covered by
`TestIsBedrockAnthropicModel` at `internal/config/load_test.go:319-320`).

The 1M annotation in `annotateBedrockModelNames` (`load.go:127-134`) is also
sensibly defensive — guarding with `!strings.Contains(models[i].Name, "1M")`
makes it idempotent if the upstream catwalk catalog ever starts shipping the
suffix natively.

Two nits:

1. The `Contains(".anthropic.")` predicate will also accept exotic prefixes like
   `foo.anthropic.bar` that AWS doesn't publish today. In practice that's fine
   — Bedrock will reject the model at invocation — but tightening to a regex
   like `^([a-z]{2}\.)?anthropic\.` would document intent and prevent any
   future false positives if AWS ever ships e.g. a `bedrock.anthropic.*`
   namespace that means something different.
2. The `prepared.Models = annotateBedrockModelNames(p.Models)` assignment at
   `load.go:303` mutates the slice elements in place via the loop in
   `annotateBedrockModelNames`. If `p.Models` is shared with another caller
   they'd observe the mutated `Name`. A one-line copy
   (`models := append([]catwalk.Model(nil), models...)` at the top of the
   helper) would make ownership explicit. Probably not load-bearing today.

Tests cover the happy paths well. Ship after either nit is addressed (or
acknowledged).
