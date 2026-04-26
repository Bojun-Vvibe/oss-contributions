# block/goose #8843 — fix(bedrock): handle ReasoningContent blocks gracefully

- **Repo**: block/goose
- **PR**: #8843
- **Author**: fresh3nough (fre$h)
- **Head SHA**: 72087c5b4b8dc437a2ecd3584cc0eed73e7172b7
- **Link**: https://github.com/block/goose/pull/8843
- **Size**: ~470 diff lines in `crates/goose/src/providers/formats/bedrock.rs`.

## What it changes

Three intertwined fixes for Bedrock's `ContentBlock::ReasoningContent`
variant, which `from_bedrock_content_block` previously swept into
`_ => bail!("Unsupported content block type from Bedrock")` and
aborted the entire response on:

1. **Outbound: `to_bedrock_message_content`
   (`bedrock.rs:55-89`)** — `MessageContent::Thinking(_)` and
   `MessageContent::RedactedThinking(_)` no longer become empty
   `ContentBlock::Text("")`. They now build proper
   `ContentBlock::ReasoningContent` blocks:
   - `Thinking` → `ReasoningTextBlock` with text + signature
     (signature only set if non-empty, matching the
     builder's optionality contract).
   - `RedactedThinking` → `RedactedContent(Blob)` after
     `BASE64_STANDARD.decode(&redacted.data)`. If decoding
     fails (because some non-Anthropic providers store the
     payload as opaque bytes, not base64), the code emits a
     `tracing::warn!` and falls back to dropping the block as
     empty text — preserving the pre-existing skip behaviour
     for cross-provider conversation histories rather than
     bailing out the whole request.
2. **Inbound: `from_bedrock_content_block`
   (`bedrock.rs:303-358`)** — return type changes from
   `Result<MessageContent>` to `Result<Option<MessageContent>>`.
   Known content-bearing variants Goose can't represent now
   error explicitly (fail fast); only the SDK's
   `non_exhaustive`-introduced `Unknown` arm returns
   `Ok(None)`. The `from_bedrock_message` caller
   (`:303-313`) collects via `.filter_map(|r| r.transpose())`
   so the `None`s drop and the `Err`s propagate.
3. **Replay-fidelity contract**: the comments at
   `:58-67` (Thinking) and `:69-89` (RedactedThinking) are
   the new normative documentation: Bedrock reasoning models
   require unmodified replay of reasoning text + signature on
   subsequent Converse calls; encrypted RedactedThinking
   payloads must be replayed as the original bytes (so we
   base64-decode before sending, since
   `from_bedrock_reasoning_content_block` base64-encodes
   them on the read path).

## Strengths

- **The "fail fast on known-but-unsupported, drop on unknown"
  policy is exactly the right contract** for an SDK
  `non_exhaustive` enum boundary. Without it, any new variant
  AWS adds would silently truncate goose responses; with it,
  the agent gets a hard error and can be patched, while
  unknown SDK-only future variants are gracefully ignored.
  The docstring at `:308-313` makes this explicit.
- **The base64-decode-with-warn-fallback for
  `RedactedThinking` (`:78-89`) preserves cross-provider
  history.** A user who switched from Anthropic-direct to
  Bedrock mid-conversation will have RedactedThinking blocks
  in history that aren't base64; without the fallback,
  Bedrock would 400-reject every subsequent request. The
  warn-and-drop is the right policy.
- **Signature optionality handled correctly** at `:62-65` —
  `if !thinking.signature.is_empty()` before
  `builder.signature(...)`. The `aws-sdk-bedrock`
  `ReasoningTextBlock::builder()` treats signature as
  optional, but passing an empty string would have produced a
  Bedrock-side validation error if the model required a
  non-empty signature.
- **`from_bedrock_message` `.filter_map(|r|
  r.transpose())` (`:303-313`) is the canonical Rust idiom**
  for "collect `Result<Option<T>>` into `Result<Vec<T>>`,
  dropping Nones, propagating Errs". Reads correctly.
- **Multi-provider RedactedThinking replay-fidelity is
  spelled out** in the comment block at `:69-89` — the
  difference between "Bedrock stores as base64 on read" and
  "Anthropic API treats `data` as opaque" is precisely what
  trips up cross-provider conversation transports, and now
  it's documented in the source.

## Concerns / asks

- **No tests in this diff for the new behaviours.** The PR
  appears to ship 470 lines of changed bedrock format logic
  with zero new unit tests, despite the existence of the
  `bedrock.rs` test module pattern in this crate. At a
  minimum:
  - A test asserting Thinking → ReasoningTextBlock round-
    trips (with and without signature).
  - A test asserting non-base64 RedactedThinking emits a
    warn and produces an empty Text block instead of an
    error.
  - A test asserting `from_bedrock_content_block` of an
    `Unknown` variant returns `Ok(None)` and the caller
    doesn't drop *valid* content next to it.
- **The `tracing::warn!` on every non-base64
  RedactedThinking (`:81-83`)** could be very noisy for a
  user who has a long Anthropic-origin history they're
  replaying through Bedrock. Consider rate-limiting (one
  warn per session, or a debug-level log after the first).
- **`from_bedrock_content_block` return-type change is a
  public-API break** (it's `pub fn`). Any downstream crate
  importing it for direct use will fail to compile after
  this PR. Worth either a `#[deprecated]`-tagged shim or a
  CHANGELOG note. If the function's only caller is
  `from_bedrock_message` in the same module, consider
  making it `pub(crate)` instead — less surface area, no
  break.
- **The error vs. None distinction depends on the SDK's
  enum membership at compile time.** If `aws-sdk-bedrock`
  adds a new variant called `Citation` and bumps in a
  minor-version update, the `_ => bail!` arm would catch
  it as an error (correct under the new policy), but a
  user upgrading the SDK would see all responses
  containing citations fail. Add a CI check that pins the
  SDK version, or wrap the error path in a feature flag.
- **`RedactedThinking` base64 fallback warns about
  "non-base64", but the actual error reason (decoder
  variant)** isn't preserved beyond `{}`. For ops triage,
  the variant name (e.g. `InvalidPadding` vs.
  `InvalidByte`) would help. `tracing::warn!(error = %err,
  "Dropping ...")` is one structured-field away.

## Verdict

**merge-after-nits** — the policy is correct, the comments
are excellent (rare for low-level SDK adapter code), and
the cross-provider replay fidelity is a genuine improvement
over the previous "abort the response" behaviour. Blockers
to land: at least two unit tests (Thinking round-trip and
non-base64 RedactedThinking fallback), and a decision on
the public-API break of `from_bedrock_content_block`'s
signature.

## What I learned

`non_exhaustive` enums at SDK boundaries demand explicit
"known-but-unsupported vs. unknown-future" policy, and the
right shape for that is `Result<Option<T>>` rather than
either `Result<T>` (silently bails on unknown) or
`Option<T>` (silently drops known errors). The
`.filter_map(|r| r.transpose())` collector is the
canonical idiom for the result type. The harder lesson is
that opaque-payload fields like RedactedThinking's `data`
have *provider-dependent* encodings (base64 here, raw
bytes there), and any cross-provider transport needs an
explicit bridge — silently dropping one side's encoding is
a worse failure than the bridge logic itself.
