# block/goose #8857 — Preserve thinking content for providers that require it

- URL: https://github.com/block/goose/pull/8857
- Head SHA: `95a5facfa1d731c163b41f2703a8abf8d3342adb`
- Size: +885 / -25 across 15 files
- Verdict: **merge-after-nits**

## What the change does

Closes #7363. Several provider families (Z.AI / Zhipu, DeepSeek, Moonshot,
Novita, Nvidia NIM, Tensorix, OpenCode-Go) require the `thinking` /
`reasoning_content` block to be **preserved on every prior assistant turn
in the conversation history** when reasoning mode is active — not just on
the latest one. Goose's existing format paths stripped or didn't surface
that content, leading to multi-turn reasoning collapse. This PR adds a
declarative `preserves_thinking: bool` flag and threads it through both
the OpenAI-compatible and Anthropic-compatible format paths.

### Declarative config

- `crates/goose/src/config/declarative_providers.rs:85-86` adds
  `preserves_thinking: bool` to `DeclarativeProviderConfig` with
  `#[serde(default, skip_serializing_if = "is_false")]` (additive,
  default `false`, omitted from JSON when `false` — backwards-compatible
  for every existing declarative provider file).
- `is_false` helper at `:91-93` is a one-line `!*value`, the canonical
  shape for `skip_serializing_if`.
- New built-in providers:
  - `crates/goose/src/providers/declarative/zai.json` — Z.AI Anthropic-
    compatible, `preserves_thinking: true`, env-var-based base URL
    (`${ZAI_BASE_URL}` default `https://api.z.ai/api/anthropic`)
  - `crates/goose/src/providers/declarative/opencode_go.json` — OpenCode
    Go, `preserves_thinking: true`, dynamic models, base URL
    `https://opencode.ai/zen/go/v1`
- Existing declarative providers (deepseek, moonshot, novita, nvidia,
  tensorix, zhipu) flip `preserves_thinking: true` — their JSON files
  each gain a one-line `+1` to opt into the new behaviour.

### Anthropic-compatible path

- `crates/goose/src/providers/anthropic.rs:60-62` adds
  `format_options: AnthropicFormatOptions` to `AnthropicProvider`
  (with `#[serde(skip)]` so it's not part of the persisted config —
  derived from `preserves_thinking` instead).
- `:154-160` adds `format_options_for_provider(preserves_thinking)`
  helper that returns `{ preserve_unsigned_thinking, preserve_thinking_context }`
  both gated on the same flag.
- `:323-329` swaps the call from `create_request(...)` to
  `create_request_with_options(..., self.format_options)` so the
  Anthropic format path receives the per-provider preservation policy.

## What is load-bearing

- `preserves_thinking` is **per-provider declarative**, not per-request
  or per-model. That's the right granularity — a provider either
  requires preserved thinking history or it doesn't, and asking the
  user to set it per-request would be a footgun.
- The Anthropic format options struct is `Copy` (implied by `#[serde(skip)]`
  and inline construction at `:154`), letting the call site pass it by
  value — clean.
- Splitting the preservation into two flags
  (`preserve_unsigned_thinking`, `preserve_thinking_context`) at
  `:155-159` means the Anthropic format layer can distinguish "preserve
  the thinking block in this turn" from "preserve thinking context across
  turns". Both flip together for now, but the two-knob shape leaves
  room to refine if a provider only needs one.

## Test coverage

- `:608` regression that the **default** is `false` — keeps every
  existing provider untouched if their JSON doesn't add the field.
- `:611-635` `test_zai_json_deserializes` pins every field on the new
  Z.AI provider including `preserves_thinking: true`, env-var name,
  base URL default, fast model, and first model name. That's a complete
  spec for the new declarative entry.
- `:638-672` `test_openai_reasoning_provider_json_preserves_thinking`
  iterates the six existing OpenAI-engine providers and asserts they
  all parse with `preserves_thinking: true` — guards against a future
  PR accidentally flipping one back to `false`.
- `:694-708` `test_opencode_go_json_deserializes` pins the second new
  provider end-to-end.

## Nits

1. **`format_options` carries a default in `new()` at `:90` but a
   computed value in `from_config()` at `:106`**. This means
   `AnthropicProvider::new()` (used in tests / direct construction)
   silently uses `preserve_*: false` regardless of config, while
   `from_config()` uses the configured value. If `new()` is only ever
   used for the default Anthropic prod provider (which doesn't need
   preservation), that's fine — but it should be commented at `:90`
   so a future caller doesn't expect `new()` to read provider config.
2. **`opencode_go.json` `preserves_thinking: true` claim**: the OpenAI-
   format path will need the symmetric handling
   (`crates/goose/src/providers/formats/openai.rs` got +408 in this
   PR, presumably implementing that, but I cannot see the full diff in
   the bounded fetch). Reviewer should verify the OpenAI-format streaming
   path actually consults `preserves_thinking` — the test at `:638-672`
   only asserts the **field deserializes**, not that the format layer
   uses it. Recommend a streaming-roundtrip test in
   `formats/openai.rs` that runs a 3-turn conversation with
   `preserves_thinking: true` and asserts each prior assistant message
   in the request payload retains its `reasoning_content`.
3. **Z.AI `api_key_env: "ZHIPU_API_KEY"`** at `zai.json` shares the env
   var with Zhipu. That's intentional (same vendor, same key) but
   surprising for a user who configured `ZHIPU_API_KEY` for Zhipu and
   suddenly finds Z.AI also picking it up. A one-line note in the
   provider's `setup_steps` would prevent confusion.

## Risk

- Additive `serde(default)` field means every existing custom provider
  config file deserializes unchanged with `preserves_thinking: false`.
- `skip_serializing_if = "is_false"` keeps round-tripped JSON byte-equal
  to original for unaffected providers — no spurious `"preserves_thinking":
  false` lines in user config files.

## Recommendation

Merge after (a) commenting that `AnthropicProvider::new()` ignores
provider-config preservation and (b) adding the OpenAI-format streaming
roundtrip test that proves the new flag actually shapes the multi-turn
request payload. The declarative wiring is solid; the OpenAI-format
behaviour needs an end-to-end pin.
