# Review: BerriAI/litellm #27115

- **Title:** feat(proxy): add health_check_reasoning_effort for model health checks
- **Head SHA:** `32a5e77adf632da7018c525dd8213e40473339f5`
- **Scope:** +45 / -0 across 2 files
- **Drip:** drip-339

## What changed

Adds a `health_check_reasoning_effort` knob so health-check pings against
reasoning-class models can specify the effort level (matches the existing
`health_check_max_tokens` pattern). Includes a new test file
`tests/test_litellm/proxy/test_health_check_max_tokens.py` (+34).

## Specific observations

- `litellm/proxy/health_check.py` (+11 / -0): plumbs the new field through
  the same call site as `health_check_max_tokens`. Confirm the field is
  read with the same precedence (env var → router config → request body)
  used by the max-tokens equivalent so users don't get surprising
  override semantics.
- `tests/test_litellm/proxy/test_health_check_max_tokens.py` (+34 / -0):
  the new test is appended to the *max_tokens* test module rather than
  living in its own `test_health_check_reasoning_effort.py`. Cosmetic, but
  future grep for the feature name will miss it. Consider renaming the
  module or splitting.
- The valid set for `reasoning_effort` is provider-specific (OpenAI:
  `low|medium|high|minimal`; Anthropic uses different shape). The PR
  appears to pass it through verbatim — call out in docs that callers are
  responsible for provider-appropriate values, or validate against an
  allowlist.
- No docs update for `health_check_reasoning_effort` under
  `docs/my-website/docs/proxy/health.md` (or similar). Health-check config
  docs are the first place ops teams look.
- Net-new code is +45 with no deletions — clean additive change, no
  behaviour shift for existing deployments that don't set the new field.

## Risks

- Silent passthrough of an invalid `reasoning_effort` could turn health
  checks red against providers that 400 on unknown values.
- Otherwise low-risk additive feature with reasonable test coverage.

## Verdict

**merge-after-nits** — split or rename the test module, add a doc snippet,
and either validate the value or call out the passthrough in docs.
