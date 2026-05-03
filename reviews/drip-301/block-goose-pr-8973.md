# block/goose PR #8973 — Improvements to LM Studio declarative provider

- Author: monroewilliams
- Head SHA: `28eb99893553c0a0347c584cef7312536c5a0b27`
- Diff: +12 / -1, single file `crates/goose/src/providers/declarative/lmstudio.json`
- Relates to #8180

## Observations

1. **`lmstudio.json:7` changes `base_url`** from the hard-coded `http://localhost:1234/v1/chat/completions` to `${LMSTUDIO_HOST}/v1/chat/completions`. Standard env-var interpolation pattern that the declarative-provider loader already supports (used by `llama_swap.json`, which the author explicitly templated against). The path suffix `/v1/chat/completions` stays hardcoded — sensible, since LM Studio's OpenAI-compat endpoint shape is fixed.
2. **`lmstudio.json:8-17` adds `env_vars`** with a single `LMSTUDIO_HOST` entry: `required: false`, `secret: false`, `primary: true`, `default: "http://localhost:1234"`. The `primary: true` flag is the right call — this is the one variable users will reach for first when pointing goose at a remote LM Studio instance. Default preserves the prior behavior exactly, so existing local-only users see no change.
3. **`lmstudio.json:18` flips `dynamic_models` to `true`**. LM Studio does expose an OpenAI-compatible `/v1/models` endpoint, so this is factually correct — and aligns with the user-visible benefit (the GUI / `goose configure` can list installed models instead of forcing manual entry). This is the higher-impact half of the PR despite being a one-line change.
4. **Trailing `models: []`** is left in place. With `dynamic_models: true`, the static array becomes a fallback / seed; worth confirming with a maintainer whether it should be removed entirely or kept as `[]` to satisfy schema requirements. Doesn't block.
5. **No test coverage** — but this is a JSON config asset for a declarative provider, and goose's provider tests are typically integration-style against real endpoints. Author's manual verification against both localhost and remote LM Studio instances (per PR body) is appropriate for this surface.

## Verdict: `merge-as-is`

Tiny, factually correct config change that mirrors the existing `llama_swap.json` pattern, preserves backward compatibility via the default, and unlocks both remote-host and dynamic-model-discovery use cases with a single env var. The `${LMSTUDIO_HOST}` pattern is already established elsewhere, so risk is essentially zero.
