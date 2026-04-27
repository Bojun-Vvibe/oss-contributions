# block/goose#8843 — fix(bedrock): handle ReasoningContent blocks gracefully

- **Head**: `0a00fa9fff733e590b59bc5cd8787695e1288a64`
- **Size**: +413/-13 in `crates/goose/src/providers/formats/bedrock.rs`
- **Verdict**: `merge-as-is`

## Context

Fixes aaif-goose/goose#8751. The Bedrock `ContentBlock` enum has a `ReasoningContent` variant that some models (e.g. `openai.gpt-oss-safeguard-120b` and other gpt-oss models on Bedrock) emit alongside their text. The previous `from_bedrock_content_block` had a catch-all `_ => bail!("Unsupported content block type from Bedrock")` arm, which meant the *entire response* was aborted as soon as any reasoning block appeared — making those models effectively unusable through Bedrock from goose.

## Design analysis

Three deltas, all correct:

### 1. Inbound: `ReasoningContent → MessageContent::{Thinking, RedactedThinking}`

`from_bedrock_content_block` now matches `ReasoningContent` explicitly at `bedrock.rs:362-364` and dispatches through a new `from_bedrock_reasoning_content_block` helper at `:392-413`:

```rust
match reasoning {
    bedrock::ReasoningContentBlock::ReasoningText(text_block) => {
        let signature = text_block.signature.clone().unwrap_or_default();
        Some(MessageContent::thinking(text_block.text.clone(), signature))
    }
    bedrock::ReasoningContentBlock::RedactedContent(blob) => {
        let encoded = base64::prelude::BASE64_STANDARD.encode(blob.as_ref());
        Some(MessageContent::redacted_thinking(encoded))
    }
    other => {
        tracing::warn!("Skipping unknown Bedrock ReasoningContent variant: {:?}", other);
        None
    }
}
```

`signature` is correctly treated as `Option<String>` with `unwrap_or_default()` — the unit test `test_from_bedrock_content_block_reasoning_text_without_signature` (PR body confirms it exists) pins this. Encrypted reasoning is base64-encoded for transport in Goose's `MessageContent::RedactedThinking`, matching how other providers in this codebase surface redacted reasoning.

### 2. Outbound: `MessageContent::Thinking → ReasoningContentBlock::ReasoningText` (replay)

Previously `to_bedrock_message_content` mapped both `Thinking` and `RedactedThinking` to `bedrock::ContentBlock::Text("".to_string())` — a silent drop. The fix at `:58-94` re-emits them faithfully:

```rust
MessageContent::Thinking(thinking) => {
    let mut builder = bedrock::ReasoningTextBlock::builder().text(&thinking.thinking);
    if !thinking.signature.is_empty() {
        builder = builder.signature(&thinking.signature);
    }
    bedrock::ContentBlock::ReasoningContent(bedrock::ReasoningContentBlock::ReasoningText(
        builder.build()?,
    ))
}
```

This is critical for multi-turn correctness. Bedrock reasoning models require the original signed reasoning text be replayed in subsequent `Converse` calls; dropping it as `Text("")` breaks the contract. The `if !thinking.signature.is_empty()` guard correctly distinguishes "we have a real signature" from "we synthesized this Thinking from a signatureless source".

The `RedactedThinking` outbound at `:74-94` correctly base64-decodes before re-wrapping (since `from_bedrock_reasoning_content_block` base64-encoded on the way in), with a graceful tracing-warn fallback if the payload isn't decodable as base64:

```rust
match base64::prelude::BASE64_STANDARD.decode(&redacted.data) {
    Ok(bytes) => bedrock::ContentBlock::ReasoningContent(
        bedrock::ReasoningContentBlock::RedactedContent(aws_smithy_types::Blob::new(bytes)),
    ),
    Err(err) => {
        tracing::warn!(
            "Dropping non-base64 RedactedThinking content when sending to Bedrock: {}",
            err
        );
        bedrock::ContentBlock::Text("".to_string())
    }
}
```

The doc comment at `:67-73` explicitly notes that other providers (Anthropic API) treat `data` as opaque, justifying the cross-provider fallback. Right call.

### 3. Signature change: `Result<MessageContent>` → `Result<Option<MessageContent>>`

The outer signature change at `:347` is the right way to express "this block is a known SDK variant we don't surface to the caller, but it isn't an error" cleanly. Caller updated at `:316` with `.filter_map(|result| result.transpose())` — that's the canonical Rust idiom for `Iterator<Result<Option<T>>> → Iterator<Result<T>>`.

Equally important: the `_ =>` catch-all is replaced by **explicit listing** of every other Bedrock variant at `:382-388` plus an `other =>` non-exhaustive arm at `:389-399`:

```rust
bedrock::ContentBlock::Audio(_)
| bedrock::ContentBlock::CitationsContent(_)
| bedrock::ContentBlock::Document(_)
| bedrock::ContentBlock::GuardContent(_)
| bedrock::ContentBlock::Image(_)
| bedrock::ContentBlock::SearchResult(_)
| bedrock::ContentBlock::Video(_) => bail!(
    "Unsupported Bedrock content block type: {}",
    bedrock_content_block_kind(block)
),
other => {
    bail!(
        "Unrecognised Bedrock content block variant: {}",
        bedrock_content_block_kind(other)
    );
}
```

This is exactly the right shape: the previous catch-all hid the SDK-version-vs-codebase-knowledge gap; the new explicit list fails the build if/when the SDK adds a new variant *and* we update the SDK without thinking, which is what you want. The `bedrock_content_block_kind` helper at `:421-437` deliberately returns only variant names (no payload) so logs can't leak model output.

## Tests

Four new unit tests called out in the PR body:

- `test_from_bedrock_content_block_reasoning_text` — `ReasoningText` w/ text + signature → `Thinking`
- `test_from_bedrock_content_block_reasoning_text_without_signature` — empty-signature path
- `test_from_bedrock_content_block_reasoning_redacted_content` — `RedactedContent(Blob)` → `RedactedThinking` w/ base64 payload
- `test_from_bedrock_message_includes_reasoning_content` — regression for #8751: mixed message (`ReasoningContent` + `Text`) converts successfully and preserves both

15 tests pass; cargo fmt + clippy `-D warnings` clean.

## Verdict reasoning

The fix solves a real "this model is currently unusable" bug, the design is symmetric (inbound encode + outbound decode, base64 on both sides), tests cover both the happy path and the fallback paths, and the catch-all-arm replacement strictly improves the codebase's ability to detect future SDK drift. Comment block at `:67-73` is exemplary — it preempts the "why are we base64-decoding here?" question.

## What I learned

The "non-exhaustive enum variant in vendored SDK + open-coded `_ =>` catch-all in our match" is a quietly common bug pattern when wrapping AWS SDKs in particular (every AWS SDK type is `non_exhaustive`). The right answer is the one this PR uses: enumerate every variant explicitly, send unknowns to a logging fallback that names the variant, and treat the catch-all as "the SDK got newer than our codebase — fail loud." Combined with `Result<Option<T>>` to express "I parsed this fine but chose not to surface it," that's a clean pattern for any external-types-to-internal-types layer.
