# sst/opencode PR #24193 — docs: clarify bedrock inference profile setup

- **Author:** anduimagui
- **Head SHA:** e4e3b841e14ebfda18054e899136b9166b39a116
- **Files:** `packages/web/src/content/docs/config.mdx` (+26 / −0),
  `packages/web/src/content/docs/providers.mdx` (+28 / −2)
- **Verdict:** `merge-after-nits`

## What the diff does

Two docs files get expanded examples for using Bedrock application
inference profile ARNs.

- `config.mdx` lines 401–432 (new block): adds a full `opencode.json`
  example showing the custom model key (`claude-sonnet-custom`),
  `setCacheKey: true`, and the ARN-as-`id` pattern.
- `providers.mdx` lines 278–322 (replacement + extension): replaces
  the previous truncated `// ...` example with a complete one,
  adds a 3-step recipe ("find system-defined inference profile →
  create application inference profile → put ARN in
  `provider.amazon-bedrock.models.<name>.id`"), and an `aws bedrock
  create-inference-profile` shell example using
  `--model-source copyFrom=arn:aws:bedrock:...:inference-profile/
  us.anthropic.claude-sonnet-4-6`.

It also includes a `:::tip` block explaining the on-demand inference
error fallback (use the system-defined inference profile ARN as
`copyFrom` instead of a raw foundation model ARN).

## Review notes

- The "Claude-like model key" guidance at line 300 is real and
  load-bearing: cache routing keys off the model name pattern, so
  users who name their custom model `my-bedrock-thing` lose Anthropic
  prompt caching even though the underlying model is Claude. Worth
  keeping.
- The CLI example at lines 307–311 hard-codes
  `123456789012` as the account ID — fine, but the surrounding text
  doesn't say "replace with your account ID". A one-line "(replace
  the account ID and region)" comment would prevent copy-paste
  confusion.
- The `:::tip` at lines 318–321 is correct but vague: "raw foundation
  model ARN" is ambiguous. Naming the actual error message Bedrock
  returns ("on-demand throughput isn't supported", or whatever the
  exact wording is) would make this discoverable via grep when a user
  hits it.
- This PR documents the exact pattern that PR #24194 would silently
  break (custom-named models with ARN `id`). Reviewers should land
  these together or hold #24194 until it can preserve user-defined
  models.

## What I learned

The Bedrock indirection layer (foundation model → system-defined
inference profile → application inference profile) is a three-tier
thing where each tier has different ARN shapes, and the only way to
make on-demand pricing work for custom routing is to copy *from* the
system-defined inference profile rather than the foundation model
directly. Worth remembering when writing any AWS provider integration.
