# sst/opencode PR #25612 — docs: per-model context size and output limit configuration

- Author: FCCMac
- Head SHA: `fe26c7cf099fa49d98c87e2b8972e86214216f0c`
- Verdict: **merge-after-nits**

## Rationale

Pure docs addition to `packages/web/src/content/docs/models.mdx:106-130` that
documents the `limit.{context,output}` config fields for local providers
that don't auto-publish capacities. Example targets the `lmstudio` provider
serving Qwen — useful and accurate. Two nits before shipping: typo
"espeically" → "especially" at line 108, and the example model id
`qwen/qwen3.6-35b-a3b` doesn't match any released Qwen tag (Qwen3 is the
current series — likely meant `qwen/qwen3-30b-a3b` or a real LM Studio
catalog id). Closes the recurring #24559 question, so worth landing once
the typo + example id are corrected.
