# PR #8948 — feat: add NVIDIA NIM as a native Rust provider

- Repo: block/goose
- Head: `930dbdcaf5e27b0c703680f329d97c8c9a49850b`
- URL: https://github.com/block/goose/pull/8948
- Verdict: **merge-after-nits**

## What lands

Adds NVIDIA NIM (`https://integrate.api.nvidia.com/v1`, OpenAI-wire-
compatible) as a native Rust provider via `OpenAiCompatibleProvider`. The PR
is additive on top of the existing declarative `nvidia.json` from #8798 —
this version provides per-model capability flags (tool / structured-output /
vision), streaming, retries that the declarative entry can't express. Three
files, +68/-0.

## Specific findings

- `crates/goose/src/providers/nvidia.rs:24-43` follows the established
  `OpenAiCompatibleProvider` pattern (xAI, DeepSeek, NanoGPT) verbatim —
  same `ProviderDef` shape, same `ConfigKey` definitions, same
  `from_env` wiring through `BoxFuture`. No deviations from the existing
  contract.
- The `ConfigKey` declarations at `nvidia.rs:34-37` look right:
  `NVIDIA_API_KEY` is `(required=true, secret=true)` and `NVIDIA_BASE_URL`
  is `(required=false, secret=false, default=NVIDIA_API_HOST)` — exactly
  the shape needed for self-hosted NIM deployments where users want to
  override the host but still have the public default work out of the box.
- `nvidia.rs:50-58` correctly uses `get_secret` for the API key and
  `get_param("NVIDIA_BASE_URL").unwrap_or_else(|_| NVIDIA_API_HOST.to_string())`
  for the host fallback. The `unwrap_or_else` swallows the error case
  (key missing from config), which is correct here because the default
  is meaningful.
- Registry wiring at `init.rs:31,80` and `mod.rs:39` is alphabetically
  sorted, matching the file's existing convention.

## Nits to address before merge

- **Known-models list will go stale fast.** `NVIDIA_KNOWN_MODELS` at
  `nvidia.rs:11-21` hardcodes 8 models (Nemotron, Llama, DeepSeek, Kimi,
  Phi, Gemma, Qwen). NIM hosts 100+ models per the PR description and
  rotates frequently. Either point at a known-models endpoint dynamically
  (some other providers do this in `OpenAiCompatibleProvider`) or accept
  that this list will need bumping every few months — and add a comment
  saying so.
- **No tests at all.** The PR description says "follows the pattern
  established by xAI, DeepSeek" — those providers have at least
  smoke-level unit tests in their `tests/` modules. Even a single
  `#[test]` checking that `NvidiaProvider::metadata()` returns the
  expected name/host would catch the most likely regression
  (someone copy-pastes the file and forgets to rename a constant).
- The doc URL at `nvidia.rs:23` (`https://docs.api.nvidia.com/nim/reference/llm-apis`)
  was reachable when the PR was opened but NIM doc URLs are notoriously
  unstable — worth using a more durable redirect target like
  `https://build.nvidia.com/explore/discover` if such a thing exists.

## Verdict rationale: merge-after-nits

The native-provider pattern is correct and the diff is small enough to
reason about end-to-end. Missing tests is the one real cost — every other
copy-paste-style provider that ships without tests becomes a quiet
maintenance burden later. A 10-line metadata round-trip test would be
plenty. The known-models churn is more of a follow-up issue than a blocker.
