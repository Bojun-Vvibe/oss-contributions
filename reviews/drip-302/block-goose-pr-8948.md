# block/goose PR #8948 — feat: add NVIDIA NIM as a native Rust provider

- Head SHA: `930dbdcaf5e2`
- Files changed (selected):
  - `crates/goose/src/providers/init.rs` (+2)
  - `crates/goose/src/providers/mod.rs` (+1)
  - `crates/goose/src/providers/nvidia.rs` (+65, new file)

## Analysis

Adds NVIDIA NIM (`https://integrate.api.nvidia.com/v1`) as a first-class native provider, layered on top of the existing `OpenAiCompatibleProvider`. NIM is OpenAI-wire-compatible, so the implementation is thin — most of the work is metadata wiring.

Registration:
- `crates/goose/src/providers/mod.rs:39` — adds `pub mod nvidia;`.
- `crates/goose/src/providers/init.rs:31` — imports `NvidiaProvider` from the parent `super` block, alphabetically sorted between `nanogpt` and `ollama`.
- `crates/goose/src/providers/init.rs:80` — registers it via `registry.register::<NvidiaProvider>(true)`. The `true` argument matches sibling entries like `kimicode`, `nanogpt`, `ollama`; this appears to be the "supported" flag (vs. `LiteLLMProvider` which is registered with `false`). Consistent with peers.

Provider impl in `crates/goose/src/providers/nvidia.rs`:
- Constants `NVIDIA_PROVIDER_NAME = "nvidia"`, `NVIDIA_API_HOST = "https://integrate.api.nvidia.com/v1"`, default model `"nvidia/llama-3.3-nemotron-super-49b-v1.5"`, doc URL `https://docs.api.nvidia.com/nim/reference/llm-apis`.
- `NVIDIA_KNOWN_MODELS` is an 8-entry `&[&str]` covering Nemotron, Llama 3.x, DeepSeek R1, Kimi-K2, Phi-4-mini, Gemma-3-27b, and Qwen3-235b. Reasonable starter set.
- `metadata()` declares two `ConfigKey` entries: `NVIDIA_API_KEY` (secret=true, required=true) and `NVIDIA_BASE_URL` (secret=false, required=false, default = host). Default-fallback semantics match other OpenAI-compatible providers in the repo.
- `from_env()` uses the standard pattern: read secret via `config.get_secret`, allow base-URL override via `get_param`, build `ApiClient::new(host, AuthMethod::BearerToken(api_key))`, and wrap it in `OpenAiCompatibleProvider::new(name, api_client, model, "")`. The trailing empty `String::new()` corresponds to whatever the constructor's last positional arg is — likely a model-prefix or system-prompt-prefix; consistent with how peers like KimiCode wire the same call.

This is straightforward, well-isolated, and additive — no changes to transport, retry, or registry contracts.

Risks / things to look at:
- The `NVIDIA_KNOWN_MODELS` list will go stale fast (NIM publishes new endpoints frequently). A follow-up to fetch this list dynamically from NIM's `/v1/models` endpoint would prevent drift; not blocking.
- The PR body mentions "richer capability coverage than the declarative entry (per-model tool / structured-output / vision flags, streaming, retries)" but the new file does not actually declare per-model capability flags — capability matrix lives elsewhere (likely in `OpenAiCompatibleProvider` or a model-config layer). Confirm whether per-model flags need to be added in `crates/goose/src/model/` or similar for the claimed vision/tool support to be exercised.
- No unit tests in the diff. At minimum, an integration-style test that asserts the provider registers and returns the expected metadata (mirroring `nanogpt.rs` test patterns if they exist) would catch regressions if `OpenAiCompatibleProvider::new`'s signature ever shifts.
- Order of items in the `KNOWN_MODELS` constant influences UI presentation in `goose configure`; current order looks intentional (default first, then meta, then deepseek, then long-tail).
- The previous declarative JSON entry from #8798 still exists somewhere in the repo. The PR claims this is "additive on top of" it, but two providers with the same name registered in two different places could shadow or conflict. Reviewer should confirm the JSON entry is removed (or explicitly co-existing) — neither change is in this diff.

## Verdict

`merge-after-nits`

## Nits

- Add an integration test asserting `NvidiaProvider::metadata()` returns the expected `ProviderMetadata` shape (name, default model, two config keys).
- Either remove the prior declarative-JSON NIM entry or explicitly document that it is superseded; otherwise a future grep for "nvidia" finds two configs.
- Consider adding a `# Safety / Limits` comment at the top of `nvidia.rs` noting NIM's per-key rate limits so contributors don't expect OpenAI-grade throughput.
