---
pr: 8798
repo: block/goose
sha: b39ff8ccbe2bf3f7530b09fc614c397daac1649d
verdict: merge-after-nits
date: 2026-04-27
---

# block/goose #8798 — Add NVIDIA provider, and improve declarative provider UX

- **Author**: jh-block
- **Head SHA**: b39ff8ccbe2bf3f7530b09fc614c397daac1649d
- **Size**: +154/-21 across 9 files (Rust core + TS UI). Closes #8505.

## Scope

Three things in one PR: (1) add NVIDIA NIM as a declarative provider via a new `nvidia.json`, (2) extend `DeclarativeProviderConfig` with `model_doc_link` and `setup_steps` so declarative providers can override the OpenAI defaults shown in the settings UI, (3) fix `ModelConfig::with_canonical_limits` so models whose canonical `output` limit equals the `context` limit don't request the entire context window as `max_tokens` (which leaves zero room for the prompt).

## Specific findings

- `crates/goose/src/providers/declarative/nvidia.json:1-24` — single-model registration (`z-ai/glm-4.7` at 131072 context), `dynamic_models: true`, `supports_streaming: true`, sensible four-step setup walkthrough, doc link to `build.nvidia.com/models`. Standard declarative pattern.
- `crates/goose/src/config/declarative_providers.rs:80-85` — `model_doc_link: Option<String>` (with `deserialize_non_empty_string`, so `""` becomes `None`) and `setup_steps: Vec<String>` (defaults to empty). Backward-compatible default-derives, all existing `*.json` files deserialize unchanged.
- `crates/goose/src/config/declarative_providers.rs:240-242,308-310` — `create_custom_provider` and `update_custom_provider` both initialize/preserve the new fields. The `update_custom_provider` path correctly carries `existing_config.model_doc_link` and `existing_config.setup_steps` through, so admin UI edits don't wipe them. Worth verifying via test (the existing `test_groq_json_deserializes` was extended at `:597-599` to assert `model_doc_link.is_none()` and `setup_steps.is_empty()`, which proves the defaults but not the round-trip).
- `crates/goose/src/model.rs:158-165` — the canonical-limit fix: `self.max_tokens = canonical.limit.output.filter(|&output| output < canonical.limit.context).map(|o| o as i32)`. The `.filter` predicate is the meat: if the canonical metadata says `output == context` (which `models.dev` apparently does for some NVIDIA-hosted models), don't use it as `max_tokens` because that's nonsense — every prompt token would consume from the same budget as output. Falling through to `None` then lets `max_output_tokens()` use its 4096 default (asserted in the new test at `model.rs:498-510`).
- `crates/goose/src/model.rs:498-510` (`skips_canonical_output_limit_when_it_equals_context_limit`) — unit test pins behaviour: NVIDIA's `moonshotai/kimi-k2.5` has `output==context==262_144`, post-fix `max_tokens` stays `None` and `max_output_tokens()` returns `4_096`. Good.
- The `<` predicate is strict: if `output < context` use it. What about `output == context - 1` (an off-by-one report from a metadata source that conservatively subtracted nothing)? The `<` rule says "use it as max_tokens" which leaves 1 token for the prompt — still nonsense. A better predicate might be a heuristic floor like `output <= context * 9 / 10` or "output leaves at least N tokens for prompt". The current rule fixes the egregious `==` case but not the asymptotic "way too high" case. Acceptable for v1 but worth a follow-up.
- `crates/goose/src/providers/init.rs:241-275` (`test_nvidia_declarative_provider_registry_wiring`) — verifies `provider_type == Declarative`, `display_name == "NVIDIA"`, `default_model == "z-ai/glm-4.7"`, `model_doc_link` flows through, `setup_steps` non-empty, `NVIDIA_API_KEY` config key is `required + secret + primary`, and importantly **NVIDIA does not expose `OPENAI_HOST` or `OPENAI_BASE_PATH`** config keys (the "stop inheriting OpenAI config fields in the settings UI" piece from the PR body). Solid assertions.
- TS side: `ui/desktop/openapi.json` + `ui/desktop/src/api/types.gen.ts` updated for the two new fields, `ConfigContext.tsx` refreshes provider state immediately after config changes (so NVIDIA appears in the model picker without modal reopen — the fourth bullet of the PR body), and `ProviderGrid.tsx` gets a one-line tweak. Coordinated.
- The PR bundles three concerns (provider add + UX field plumbing + canonical-limit fix). The canonical-limit fix is the kind of correctness change that benefits from being its own PR with its own bisect target — splittable in retrospect.

## Risk

Low for the additive bits (new declarative provider, new optional config fields). Medium-but-positive for the canonical-limit fix: it changes `max_tokens` derivation for any model whose canonical metadata reports `output >= context`. Need to spot-check that no currently-listed provider depended on the old (broken) behaviour.

## Verdict

**merge-after-nits** — clean overall. Worth (1) splitting the canonical-limit fix into its own PR for cleaner bisect history, (2) reconsidering the `<` predicate in `model.rs:160` to handle the "output is almost as large as context" pathology not just the equality case, and (3) adding a round-trip test (write → read → assert preserved) for `model_doc_link` and `setup_steps` through `create_custom_provider`/`update_custom_provider`. Functionality is correct.
