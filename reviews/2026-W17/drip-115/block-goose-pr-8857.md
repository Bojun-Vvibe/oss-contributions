# block/goose#8857 — Preserve thinking content for providers that require it

- PR: https://github.com/block/goose/pull/8857
- Head SHA: `95a5facf`
- Diff: +885 / -25 across 15 files
- Fixes #7363

## What it does
Two parallel additions across the provider layer:

1. **Anthropic-compatible declarative providers gain a `preserves_thinking: bool` flag** (`config/declarative_providers.rs:85` field, `:91-94` `is_false` skipper, default `false`). New built-in `declarative/zai.json` (+34) wires `preserves_thinking: true` for Z.AI's GLM-4.5/5.1, which require the prior assistant turn's `thinking` block to be replayed for tool-calling continuity. The Anthropic format layer (`providers/formats/anthropic.rs` +265/-10) consumes the flag and includes thinking blocks in subsequent assistant messages instead of stripping them.

2. **OpenAI-compatible providers gain `clear_thinking: false` support** (`providers/openai.rs` +12/-2, `providers/formats/openai.rs` +408/-10). Several built-in declarative providers (`deepseek.json`, `moonshot.json`, `novita.json`, `nvidia.json`, `tensorix.json`, `zhipu.json`, `opencode_go.json` +29) get one-line additions to opt in. Also fixes a duplicate-reasoning-content bug in the streaming OpenAI-compatible code path (per PR description; concrete site likely in the `formats/openai.rs` +408 hunk).

3. Tests:
   - `test_zai_json_deserializes` (`declarative_providers.rs:609`+) — asserts the new built-in parses, has the right engine/api_key/base_url, and `preserves_thinking == true`.
   - `test_openai_reasoning_provider_json_preserves_thinking` (`:646`+) — table-driven across all six new opt-in OpenAI providers, asserts each parses and matches its declared name.
   - Anthropic format tests (+265 in `formats/anthropic.rs`) — formatting behavior with thinking blocks.
   - OpenAI streaming reasoning regression tests (+408 in `formats/openai.rs`).

## Strengths
- Right-shaped fix. The two distinct provider families need distinct handling (Anthropic threads thinking through `thinking` content blocks; OpenAI-compat threads it through `reasoning_content` in `delta`/`message`), and the PR keeps them in their respective format layers instead of bolting on a generic abstraction. That's the maintainable choice.
- Default `false` for `preserves_thinking` (`:91`) is the correct backward-compatible default — only the explicit opt-in providers get the new behavior.
- `is_false` (`:91-93`) used as `skip_serializing_if` keeps generated configs clean (no noisy `"preserves_thinking": false` everywhere).
- The table-driven test across six provider JSONs is exactly the right shape — every time someone adds a new opt-in provider, it gets the same coverage automatically by appending one tuple.
- Adding a UI/openapi schema field (`ui/desktop/openapi.json` +3, `ui/desktop/src/api/types.gen.ts` +1) means the desktop UI sees the flag too, so any future "edit provider" form can surface it.

## Concerns
- **Diff is large (+885) and touches a behavior-sensitive path.** Reasoning/thinking handling is the kind of code where a one-character bug becomes a silent quality regression that surfaces weeks later as "the model started getting confused on tool chains." The +408 in `formats/openai.rs` is the riskiest hunk — make sure CI exercises both the streaming and non-streaming reasoning paths, and that there's a test for the *duplicate reasoning content* bug (the PR mentions the fix but I can't see it in the visible diff).
- **`opencode_go.json` (+29)** ships a new declarative provider as part of this PR. The provider name is innocuous from this repo's standpoint, but bundling a new provider definition with a thinking-preservation feature PR muddies the change history. Consider splitting into (a) the `preserves_thinking` infrastructure + Z.AI built-in, (b) the OpenAI streaming reasoning fix, (c) the new opencode_go provider.
- **`zai.json` includes `${ZAI_BASE_URL}` env-var indirection** (`:609`+ test asserts default `https://api.z.ai/api/anthropic`). Good — users can repoint to a self-hosted gateway. But verify the env-var loader doesn't crash when `ZAI_BASE_URL` is unset (the test only asserts the default field is present, not that the loader handles unset).
- **Migration story**: existing users with custom declarative provider configs won't have `preserves_thinking`, which defaults to `false` via serde — that's correct backward compat. Worth a one-line release note so power users know they can opt their custom configs in.
- **No mention of the `clear_thinking: false` semantics in the OpenAI providers' user-facing docs.** The flag is added but if a user is configuring their own custom OpenAI-compatible provider, how do they know to set it? At minimum, document on the "create custom provider" UI form.

## Verdict
**needs-discussion** — feature is wanted and the structure is sound, but the bundled provider addition + 400-line streaming reasoning rewrite + thinking-preservation infrastructure should ideally land as 2–3 separate PRs. If maintainers prefer to land it as one, request a maintainer to walk through the `formats/openai.rs` streaming hunk in detail and confirm the duplicate-reasoning regression test is present.
