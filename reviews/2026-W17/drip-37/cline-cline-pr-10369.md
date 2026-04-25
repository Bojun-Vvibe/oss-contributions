# cline/cline #10369 — Strip data URI prefix from images for Ollama API compatibility

- **Repo**: cline/cline
- **PR**: [#10369](https://github.com/cline/cline/pull/10369)
- **Head SHA**: `e6da0c47e053fdf2a09a2c147d1204e04a876daf`
- **Author**: alkul93123
- **State**: OPEN (+20 / -10)
- **Verdict**: `merge-after-nits`

## Context

When using Cline against an Ollama vision model
(`qwen3-vl`, `llava`, etc.), uploaded images would be silently
ignored or rejected by the model. Root cause: Cline was inlining
images into the message `content` string as
`data:image/...;base64,<...>` Data URIs, but the Ollama API
expects raw base64 strings in a *separate* `images: string[]`
field on the message. So the model was seeing a giant blob of
text containing `data:image/png;base64,...` instead of an image.
Closes cline#10368.

## Design

Two parallel changes in `src/core/api/transform/ollama-format.ts`:

1. **Tool-result image path** (lines 46-54 of the diff): the
   existing branch that pushed
   `` `data:${part.source.media_type};base64,${part.source.data}` ``
   into `toolResultImages` is rewritten to strip any leading
   `data:[^;]+;base64,` via a regex `replace(/^data:[^;]+;base64,/, "")`
   and push the bare base64 instead. The "see following user
   message for image" placeholder text is preserved in the
   stringified content.
2. **Non-tool message image path** (lines 64-86 of the diff):
   previously this branch inlined the data-URI string into the
   joined `content`. Now there's a parallel `nonToolImages: string[]`
   collector; image parts go to that array (with the same regex
   strip), text parts go to `content`. The pushed message gains
   an `images` field set to `nonToolImages` when non-empty (or
   `undefined` to omit the key entirely from the wire message).

Regex `^data:[^;]+;base64,` correctly:

- Anchors at start of string (`^`), so it won't strip a `data:`
  substring that legitimately appears mid-base64 (which it
  wouldn't, base64 alphabet is `[A-Za-z0-9+/=]`, but the anchor
  is correct defense).
- Uses `[^;]+` for the media-type, which covers
  `image/png`, `image/jpeg`, `image/webp`, `image/gif`, etc.
- Single `replace` call without `/g` flag — only the first match
  is consumed, which is what we want.

## Risks

1. **Already-stripped base64.** If `part.source.data` is *already*
   raw base64 (no Data URI prefix), the regex no-ops and the raw
   string is pushed — correct behavior. Verified: `replace()`
   with no match returns the original string.
2. **`images: undefined` serialization.** Setting `images:
   nonToolImages.length > 0 ? nonToolImages : undefined` relies
   on `JSON.stringify` dropping `undefined` values, which it does
   for plain object properties. So a no-image message ships with
   no `images` key, matching the pre-fix wire shape. Good.
3. **Tool-result images path is asymmetric.** The tool-result
   branch still pushes into the *outer* `toolResultImages` array
   and stringifies the placeholder text into the tool-result
   content — but I don't see (in my diff window) where
   `toolResultImages` is then attached to the outgoing message
   as `images: [...]`. The PR description says the fix applies to
   "both regular user messages and tool results" — worth
   confirming the tool-result path also wires the array onto the
   subsequent user message, not just collects it. If
   `toolResultImages` is collected and never sent, the
   tool-result image fix is a no-op.

## Suggestions

- **Verify tool-result `images` wiring.** Either confirm the
  collector is consumed downstream (the snippet I have cuts off
  before showing where `toolResultImages` lands) or extract the
  same `pushImageMessage(role, content, images)` helper so both
  paths use one constructor.
- **Tighten the regex.** `^data:[^;]+;base64,` allows
  `^data:;base64,` (empty media-type). The Anthropic SDK won't
  emit that, but a stricter `^data:image\/[^;]+;base64,` matches
  the Anthropic image-block contract more closely.
- **Add a unit test** under `src/core/api/transform/__tests__/`
  asserting the wire shape: input Anthropic image block →
  output Ollama message has `images: ["<bare base64>"]` and
  `content` containing the placeholder text. The PR currently
  ships the fix without a regression test, which is the largest
  durability gap.

## What I learned

Two-channel message shapes (text in `content`, binary handles in
a sibling field) are easy to get wrong when the source SDK
encodes both channels into one stringified payload. The
transform layer is the only place to demultiplex them — and the
right transform produces a parallel array, *not* a header-only
message string with the binary inlined. The `images: undefined`
trick to omit the key entirely is the right way to keep
backward-compatible wire shape for text-only messages.
