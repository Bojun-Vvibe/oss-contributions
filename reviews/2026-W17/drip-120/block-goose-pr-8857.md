# PR #8857 — Preserve thinking content for providers that require it

- **Repo**: block/goose
- **PR**: #8857
- **Head SHA**: `95a5facf`
- **Author**: jh-block
- **Size**: +885 / -25 across 15 files
- **URL**: https://github.com/block/goose/pull/8857
- **Verdict**: **merge-after-nits**

## Summary

Several providers (DeepSeek, Moonshot, Z.AI's `glm-*` reasoning
models, Novita, NVIDIA NIM, Tensorix, Zhipu, OpenCode-Go) require
that prior-turn `<think>...</think>` reasoning blocks survive into
subsequent assistant turns or they error / silently degrade quality
on multi-turn conversations. Goose's previous behavior was to
strip thinking from history before resending. This PR adds a
declarative-provider field `preserves_thinking: bool` (default
false), turns it on for the affected providers' JSON manifests,
introduces an `AnthropicFormatOptions` carrier and a
`create_request_with_options` entry point so the Anthropic
formatter can be tuned per-provider, and threads the flag through
the Anthropic provider so the constructed request preserves
thinking content when the flag is set.

## Specific changes

- `crates/goose/src/config/declarative_providers.rs:82-95` —
  `DeclarativeProviderConfig` gains
  `pub preserves_thinking: bool` with `#[serde(default,
  skip_serializing_if = "is_false")]` so the field stays absent
  in serialized manifests when the default is in effect (preserves
  diff-cleanliness across the provider JSON files). The local
  `is_false(&bool) -> bool` helper at `:90-92` is the standard
  pattern; correct.
- `crates/goose/src/config/declarative_providers.rs:246-321` —
  `create_custom_provider` and `update_custom_provider` both
  forward the field. The update path correctly reads
  `existing_config.preserves_thinking` (preserving operator-set
  values) rather than defaulting to false on update, which would
  silently regress.
- `crates/goose/src/config/declarative_providers.rs:605-707` —
  three new test functions:
  - `test_zai_json_deserializes` — pins that `zai.json`
    advertises `preserves_thinking: true`, plus the env-var and
    catalog-id wiring.
  - `test_openai_reasoning_provider_json_preserves_thinking` —
    parametrizes over `deepseek`, `moonshot`, `novita`, `nvidia`,
    `tensorix`, `zhipu`, asserts each is OpenAI-engine and has
    `preserves_thinking: true`. This is a good guard against a
    later JSON edit accidentally flipping the field.
  - `test_opencode_go_json_deserializes` — the new
    `opencode_go.json` file is exercised.
- `crates/goose/src/providers/anthropic.rs:60-110` —
  `AnthropicProvider` gains `format_options:
  AnthropicFormatOptions` (with `#[serde(skip)]` so it doesn't
  pollute the persisted provider config — only set at construction
  time from declarative metadata). The `complete()` path at
  `:320-332` swaps the call from `create_request(...)` to
  `create_request_with_options(...)` so options reach the formatter.
- The provider JSON files (`deepseek.json`, `moonshot.json`,
  `novita.json`, `nvidia.json`, `tensorix.json`, `zhipu.json`,
  `zai.json`, `opencode_go.json`) each get
  `"preserves_thinking": true`. The `opencode_go.json` is wholly
  new (29 lines); rest are 1-line additions.

## Risks

Two structural concerns worth resolving before merge.

1. **Coupling with Z.AI breakage history**: This PR overlaps in
   intent with the upstream `qwen-code` discussion thread (#3304 /
   #3579 / #3682) about whether reasoning preservation belongs in
   *history mutation* vs *request-boundary serialization*. The Goose
   approach here is the former — preserve the canonical history with
   thinking blocks intact, gate behavior on a per-provider flag.
   That's defensible because Goose's history representation already
   carries the thinking blocks; the alternative (serialize-time
   filtering) would require touching every provider's request
   builder. The `preserves_thinking` flag name is slightly
   misleading because it describes a *requirement* of the upstream,
   not a *behavior* of Goose ("requires_thinking_in_history" or
   "round_trip_thinking" would be clearer). Worth a doc comment on
   the field.

2. **Anthropic provider is the only consumer wired**: the diff
   adds the flag to OpenAI-engine providers (deepseek, moonshot,
   novita, nvidia, tensorix, zhipu) but the implementation wiring
   only flows through `crates/goose/src/providers/anthropic.rs`
   and `formats/anthropic.rs`. The OpenAI-engine providers in
   the manifest are *advertising* the flag without anything in
   `providers/openai_compatible.rs` reading it. Either the OpenAI
   formatter needs the same `_with_options` overload (likely the
   intent for a follow-up commit in this PR) or the JSON additions
   are premature and should be deferred until the OpenAI side
   lands. From the diff context I can only see the Anthropic side
   wired, so confirm before merge that the OpenAI-compatible
   formatter is also reading `format_options.preserves_thinking`.

## Verdict rationale

The architectural shape (capability flag on declarative provider
manifest, per-engine `_with_options` formatter overload, default-
false to preserve existing behavior) is the right way to land
this. Tests pin every JSON file's flag value. Merge after
clarifying that the OpenAI-compat formatter consumes the flag or
splitting the OpenAI-side JSON additions into a follow-up PR that
lands together with the formatter wiring; and after a one-line
docstring on the field clarifying that it describes an upstream
requirement, not a Goose-side behavior toggle.
