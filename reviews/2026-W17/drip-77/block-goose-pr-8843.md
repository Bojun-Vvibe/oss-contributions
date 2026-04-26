---
pr: 8843
repo: block/goose
sha: 72087c5b4b8d
verdict: merge-after-nits
date: 2026-04-26
---

# block/goose #8843 — fix(bedrock): handle ReasoningContent blocks gracefully

- **URL**: https://github.com/block/goose/pull/8843
- **Author**: fresh3nough
- **Head SHA**: 72087c5b4b8d
- **Size**: +413/-13 in 1 file (`crates/goose/src/providers/formats/bedrock.rs`)

## Scope

Fixes two related bugs in the Bedrock Converse format:

1. **Outbound (`to_bedrock_message_content` at `:55-95`)**: `MessageContent::Thinking` and `MessageContent::RedactedThinking` were previously dropped as empty-text blocks. Now both are re-emitted as `ContentBlock::ReasoningContent` with the original text + signature for `Thinking`, and base64-decoded blob for `RedactedThinking`. This is required because Bedrock reasoning models (Claude with reasoning, `openai.gpt-oss-*`) reject subsequent Converse calls if the prior reasoning isn't replayed verbatim with its signature.

2. **Inbound (`from_bedrock_content_block` at `:316-444`)**: signature changes from `Result<MessageContent>` to `Result<Option<MessageContent>>`. Adds explicit `ReasoningContent` handling that maps `ReasoningText` → `MessageContent::thinking` and `RedactedContent` → base64-encoded `MessageContent::redacted_thinking`. Replaces the `_ => bail!("Unsupported content block type")` catchall with an explicit enumeration of `Audio | CitationsContent | Document | GuardContent | Image | SearchResult | Video` (all bail) and a separate "Unrecognised variant" arm for the SDK's `non_exhaustive` future-proofing.

3. **Caller adaptation (`from_bedrock_message` at `:303-310`)**: adds `.filter_map(|result| result.transpose())` to drop the `None` returns (currently only fires for the unknown-variant arm if it ever returns `None` — see findings).

## Specific findings

- **The base64 round-trip for `RedactedThinking` is the subtle correctness anchor.** Outbound at `:74-85` decodes the stored base64 string to bytes before sending; inbound at `:417-419` re-encodes blob bytes to base64 before storing. Symmetric. The fall-through at `:81-87` for non-decodable input drops the block with a `tracing::warn!` rather than aborting — explicitly justified in the comment as "RedactedThinking is also produced by other providers (e.g. the Anthropic API) which treat `data` as an opaque payload rather than guaranteed base64". This is the right call: cross-provider conversation history shouldn't fatal-abort a Bedrock send because some Anthropic-direct turn has non-base64 redacted bytes. Worth a follow-up: tag `RedactedThinking` content with its provenance provider so we know whether base64-decoding is meaningful.

- **`Result<Option<MessageContent>>` return type is the right shape** for "this block type exists but produces nothing" cases, but the only place that actually returns `Ok(None)` in the visible diff is... none of the explicitly enumerated arms. They all `Some(...)` or `bail!`. `from_bedrock_reasoning_content_block` returns `Option` with `None` only on the SDK's unknown reasoning variant (`:443`), and that fall-through is a `tracing::warn!` + `None`. So the outer `Option` exists for one specific case: silent-drop of unrecognized reasoning sub-variants. That's fine but worth a doc-comment on `from_bedrock_content_block` clarifying "`Ok(None)` means the SDK presented a sub-variant we deliberately chose to ignore rather than fail; `Err` means we hit a known-unhandled type or a wholly unknown top-level variant".

- **`Audio | CitationsContent | Document | GuardContent | Image | SearchResult | Video → bail!`** at `:386-397` is a deliberate fail-fast choice. The PR's docstring at `:323-329` makes the rationale explicit ("Known content-bearing variants that Goose cannot represent return an error so that the caller fails fast rather than silently truncating responses"). This is the right policy — silent truncation of, say, citation content is worse than a hard error. But: it's a behavior change. Any existing user whose Bedrock model returns `CitationsContent` or `Document` blocks (e.g. Bedrock Knowledge Bases responses, multimodal inputs) will start seeing hard failures where they previously saw a "Unsupported content block type" error. That's strictly better debuggability but may surface latent breakage. Worth a CHANGELOG entry.

- **The `other` arm at `:398-408`** uses `bedrock_content_block_kind(other)` for the error message — that helper isn't visible in the diff. Reviewer should confirm it produces a meaningful name (not `"unknown"`) for the SDK's `non_exhaustive` future-arm. If the helper is `"unknown"` for everything it doesn't recognize, the error is "Unrecognised Bedrock content block variant: unknown" which doesn't help debugging. Better: include the `Debug`-formatted variant or the discriminant byte.

- **Outbound builder error propagation at `:69`**: `builder.build()?` propagates the `aws_sdk_bedrockruntime::error::BuildError` through `?`. That should be fine because the surrounding function returns `Result<bedrock::ContentBlock>`. Confirm the error type composition lines up.

- **`builder.signature(&thinking.signature)` is conditionally added only when non-empty** (`:67-69`). Good — Bedrock's Builder pattern usually treats `signature(empty_string)` as "set to empty" rather than "leave unset", which fails server-side validation differently than "unset". The conditional is the right defensive shape.

- **`thinking.signature.is_empty()` fall-through to no-signature path**: legitimate for Anthropic-direct → Bedrock cross-provider history (where the upstream signature is unknown). For Bedrock-native reasoning, signature is always present and the conditional just becomes a no-op. Correct in both directions.

- **`from_bedrock_message` filter_map at `:306`**: `.filter_map(|result| result.transpose())` is the canonical "drop Ok(None), keep Ok(Some) and Err". Idiomatic and right.

- **Single-file 413/-13 diff in one provider format file** is appropriately scoped. No drive-by reformatting, no unrelated cleanups.

- **Tests**: none visible in the diff. The two failure modes that need regression coverage are: (a) round-trip of `MessageContent::Thinking` through `to_bedrock_message_content` → `from_bedrock_content_block` preserves text and signature; (b) `Audio` block in a Bedrock response now produces a hard error rather than a silent drop. Both are 10-line tests against literal bedrock SDK structs. Strongly recommend adding before merge.

- **`base64::prelude::BASE64_STANDARD` import** is fine; consider whether `BASE64_STANDARD_NO_PAD` is what Bedrock actually uses on the wire (usually no-pad on URL contexts, padded on JSON). If Bedrock wraps the base64 in a JSON string field, padded standard is correct.

## Risk

Medium. Two failure-mode flips in one PR: (a) reasoning is now preserved across turns instead of silently dropped — *strictly better* for reasoning-model UX, no regression risk; (b) unknown content types now hard-error instead of falling through — *strictly better debuggability*, but may surface latent unhandled-content-type bugs in user workflows that were silently broken before. (b) is the higher risk surface and the one a CHANGELOG note should call out.

## Nits

1. Add tests: `Thinking` round-trip + `Audio` hard-error.
2. Confirm `bedrock_content_block_kind(other)` produces a meaningful name for SDK-future variants.
3. Doc-comment `from_bedrock_content_block` with the precise meaning of `Ok(None)` vs `Err`.
4. Add CHANGELOG entry: "Bedrock provider now hard-errors on `Audio | CitationsContent | Document | GuardContent | Image | SearchResult | Video` content blocks instead of dropping with a generic message — please file an issue if you need any of these handled."
5. Confirm `BASE64_STANDARD` (padded) matches Bedrock's wire format for `RedactedContent`.
6. Follow-up: tag `RedactedThinking::data` with originating provider so cross-provider history can be base64-validated only where it's meaningful.

## Verdict

**merge-after-nits** — closes a real bug (reasoning models break on multi-turn after a `Thinking` block was dropped) with the right Bedrock SDK shape, and tightens the unknown-content-type handling correctly. The base64 round-trip symmetry is correct and the cross-provider compatibility fall-through is well-considered. Test coverage and the changelog entry are the gating asks.

## What I learned

Bedrock's `non_exhaustive` Rust enums are a forcing function for "explicit list every variant + named other arm", and that pattern (versus the catch-all `_ =>`) is the difference between "we silently drop tomorrow's SDK addition" and "we fail loudly when the SDK adds something we haven't audited". Reasoning content specifically is the kind of thing where the model's correctness depends on the client *not* round-tripping through a lossy representation — the signature isn't decorative, it's load-bearing for the upstream reasoning replay protocol. Any provider format that handles reasoning models needs the same outbound-replay + inbound-preserve symmetry this PR establishes for Bedrock.
