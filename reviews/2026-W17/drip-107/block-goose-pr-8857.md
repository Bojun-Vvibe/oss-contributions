# block/goose #8857 — Add declarative Anthropic thinking preservation support

- **Repo**: block/goose
- **PR**: #8857
- **Author**: jh-block
- **Head SHA**: a02a417871eb7334d180990abc44ed72253dec0b
- **Base**: main
- **Size**: +505 / −15 across seven files. Largest blocks:
  `crates/goose/src/providers/formats/anthropic.rs` (+265/-10),
  `crates/goose/src/providers/formats/openai.rs` (+125/-3),
  `crates/goose/src/providers/anthropic.rs` (+43/-2),
  `crates/goose/src/config/declarative_providers.rs` (+34),
  new `crates/goose/src/providers/declarative/zai.json` (+34),
  `ui/desktop/openapi.json` (+3) and `ui/desktop/src/api/types.gen.ts` (+1).

## What it changes

Adds a declarative `preserves_thinking` flag for Anthropic-engine
providers (so JSON-defined providers can opt in to "preserve
upstream thinking blocks across turns" without a Rust code
change), and ships a built-in Z.AI provider that uses it.

Three layers:

1. **Config surface** (`declarative_providers.rs:82-95`):
   `DeclarativeProviderConfig` gains a
   `#[serde(default, skip_serializing_if = "is_false")]`
   `preserves_thinking: bool` field with helper `is_false` at
   `:91-93`. Default `false` keeps the existing strict-Anthropic
   behavior. The `create_custom_provider` and
   `update_custom_provider` paths at `:240-318` carry the field
   through.
2. **Format options plumbing** (`formats/anthropic.rs:42-68`): a
   new `AnthropicFormatOptions { preserve_unsigned_thinking,
   preserve_thinking_context }` struct (Copy, Default, Eq). The
   `for_model` resolver at `:48-67` lets a per-model
   `request_params.preserve_thinking_context` /
   `ANTHROPIC_PRESERVE_THINKING_CONTEXT` env var override the
   provider-level flag, and *forces* `preserve_unsigned_thinking
   = true` whenever `preserve_thinking_context` is true (the
   `... || preserve_thinking_context` at `:62`). That's the
   right invariant — preserved context without preserved
   unsigned-thinking blocks would be incoherent.
3. **Wire format** (`formats/anthropic.rs:138-260`,
   `:519-680`, `:755-770`):
   - `format_messages_with_options` at `:144-260` forwards the
     options down to the per-message walk; the existing
     `format_messages` becomes a thin wrapper that passes
     `Default::default()`.
   - Inside the walk at `:225-237`, `MessageContent::Thinking`
     blocks with empty signatures used to be dropped silently;
     under `preserve_unsigned_thinking` they're emitted as
     `{type: "thinking", thinking: <text>}` (no signature
     field).
   - `response_to_message` at `:395-400` no longer hard-fails
     on missing signatures (`ok_or_else(... "Missing thinking
     signature")` → `unwrap_or_default()`), and the streaming
     path at `:755-770` reads the initial thinking text /
     signature out of the content-block-start payload instead
     of starting with empty strings.
   - `apply_thinking_config` at `:545-585` is restructured to
     route the `budget_tokens` derivation through a new
     `thinking_budget_tokens` helper at `:519-540` (which now
     also reads `ANTHROPIC_THINKING_BUDGET` in addition to
     `CLAUDE_THINKING_BUDGET`), and when
     `preserve_thinking_context` is set it injects
     `clear_thinking: false` into the `thinking` block — and,
     if no `thinking` block is present, synthesizes one with
     `enabled` + the resolved `budget_tokens`.
4. **Provider wiring** (`providers/anthropic.rs:58-167`):
   `AnthropicProvider` gains a `#[serde(skip)] format_options:
   AnthropicFormatOptions` field. `Self::format_options_for_provider`
   at `:153-160` derives it from the provider config's
   `preserves_thinking`. The `Provider::stream` impl at `:320-330`
   now calls `create_request_with_options(...,
   self.format_options)` instead of the plain `create_request`.
5. **Built-in Z.AI provider** (`declarative/zai.json:1-34`):
   `engine: "anthropic"`, `base_url:
   "${ZAI_BASE_URL}"` (defaulting to
   `"https://api.z.ai/api/anthropic"`), `api_key_env:
   "ZHIPU_API_KEY"`, `preserves_thinking: true`, and the GLM
   model lineup (glm-5.1, glm-5, glm-5-turbo, glm-4.7,
   glm-4.7-flash, glm-4.7-flashx, glm-4.6, glm-4.5,
   glm-4.5-air, glm-4.5-flash) with their context limits.

## Strengths

- The decision to make the flag declarative (not a per-provider
  Rust impl) is right. Z.AI is "Anthropic wire format with
  different signing semantics" — adding a Rust subclass would
  duplicate the entire format module. The
  `AnthropicFormatOptions` struct gives future
  Anthropic-compatible providers a single switch to flip.
- The `preserve_unsigned_thinking || preserve_thinking_context`
  invariant at `formats/anthropic.rs:60-62` enforces the only
  coherent combination. A user who sets only
  `preserve_thinking_context` would otherwise get a request
  with `clear_thinking: false` but with thinking blocks
  silently stripped from prior turns — incoherent. Forcing
  the unsigned-preservation flag closes that hole.
- `response_to_message` at `:395-400` softening from
  `ok_or_else(anyhow!("Missing thinking signature"))` to
  `unwrap_or_default()` is the right call for
  non-canonical-Anthropic providers that don't emit
  signatures, and the new streaming path at `:755-770` makes
  the same accommodation symmetrically. The
  parse-side and emit-side both relax in lockstep.
- The `is_false` helper + `skip_serializing_if` at
  `declarative_providers.rs:91-93` keeps existing
  on-disk provider JSON files unchanged when the flag is
  false (which is the default). No churn on user-facing
  config files.
- The `format_options_for_provider` test pair at
  `providers/anthropic.rs:368-388` pins both default and
  preserve cases. The `test_zai_json_deserializes` test at
  `declarative_providers.rs:610-635` pins the bundled JSON
  schema (env var name, default base URL, fast model,
  `preserves_thinking: true`, model list).
- Splitting `thinking_budget_tokens` out of
  `apply_thinking_config` (`anthropic.rs:519-540`) makes the
  budget-resolution logic testable in isolation and also DRYs
  up the new `preserve_thinking_context` synthesis path
  (`:573-585`) so the budget number agrees on both sides.
- The new `ANTHROPIC_THINKING_BUDGET` env var is added
  *alongside* `CLAUDE_THINKING_BUDGET` (resolution at
  `:534-538` checks both), preserving backcompat.

## Risks / nits

- `clear_thinking: false` injection at `anthropic.rs:584-585`
  is unconditional once `preserve_thinking_context` is on,
  even when `apply_thinking_config` was about to set
  `ThinkingType::Adaptive` at `:548-553`. The adaptive arm
  doesn't insert a `thinking` block — it inserts
  `output_config: {effort}`. So when both adaptive thinking
  and `preserve_thinking_context` are active, the
  `obj.get_mut("thinking")` at `:584` will be `None` and the
  insert at `:573-583` will fire, producing a request with
  *both* `output_config: {effort: ...}` and
  `thinking: {type: "enabled", budget_tokens: ..., clear_thinking:
  false}`. Anthropic's wire format may not accept both; needs
  a test to confirm. If it doesn't, the `preserve_thinking_context`
  path needs to early-return when adaptive is selected, or
  rewrite `output_config` instead.
- Dropping the `Missing thinking signature` error at `:398`
  weakens canonical-Anthropic validation. Real Anthropic
  responses *should* always carry a signature for thinking
  blocks; silently substituting empty strings hides spec
  drift on Anthropic itself, not just on Z.AI. Worth
  considering: gate the `unwrap_or_default()` on the
  effective `preserve_unsigned_thinking` (or on a per-provider
  "is canonical Anthropic" boolean) so the Z.AI relaxation
  doesn't degrade Anthropic's own error reporting.
- `AnthropicFormatOptions` is `#[serde(skip)]` on
  `AnthropicProvider` (`providers/anthropic.rs:62`). That's
  fine for *new* construction (it's recomputed in
  `Self::from_config`), but if a provider instance is ever
  reconstructed from a serialized form via the inventory
  path (`inventory.rs`), `format_options` will silently
  default to `AnthropicFormatOptions::default()` — i.e.,
  preserve flag *off*, even if the underlying declarative
  config had `preserves_thinking: true`. Worth a regression
  test for "round-trip the inventory and re-derive
  format_options from the declarative config".
- The `ANTHROPIC_PRESERVE_THINKING_CONTEXT` env var
  (`anthropic.rs:53-58`) is documented only by its presence
  in the `for_model` resolver. Consider adding a comment
  pointing at user docs, and listing it (alongside
  `ANTHROPIC_PRESERVE_UNSIGNED_THINKING` and
  `ANTHROPIC_THINKING_BUDGET`) in whatever
  env-var-reference page the project keeps.
- 505 lines is the upper edge of "single-PR feature". The
  Z.AI bundled JSON could land separately from the
  format-options refactor; that would let the format-options
  refactor merge first (with no new provider) and the Z.AI
  drop in a one-file follow-up. Not a blocker — the tests
  cover both — but easier to bisect later.

## Suggestions

- Add a test for `preserve_thinking_context` *combined with*
  `ThinkingType::Adaptive` to pin the wire format the
  combination produces, or document that the combination is
  unsupported and reject it at config-load time.
- Make the response-side `unwrap_or_default()` for missing
  thinking signatures (`anthropic.rs:398`) gated on
  `format_options.preserve_unsigned_thinking`, so canonical
  Anthropic still surfaces the missing-signature drift as a
  hard error.
- Add a round-trip test that constructs an
  `AnthropicProvider` from a declarative config with
  `preserves_thinking: true`, serializes it through the
  inventory layer, deserializes, and checks that
  `format_options.preserve_thinking_context` is still
  `true` (or fix the `#[serde(skip)]` to derive on load).
- Document the three env vars
  (`ANTHROPIC_PRESERVE_UNSIGNED_THINKING`,
  `ANTHROPIC_PRESERVE_THINKING_CONTEXT`,
  `ANTHROPIC_THINKING_BUDGET`) in one place.
- Consider splitting future Anthropic-compatible provider
  drops (Z.AI plus whatever's next) from the format-options
  refactor in subsequent PRs.

## Verdict

`merge-after-nits` — the design (declarative flag, format-options
struct, two-axis preservation with the `||` invariant) is
right, and the test coverage on the format paths and the
bundled JSON is solid. The adaptive-thinking + preserve-context
interaction and the `#[serde(skip)]` round-trip risk are the
two material asks before merging.
